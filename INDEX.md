# Skill Index

**651 curated skills** organized by category. Each skill has been individually reviewed by Claude for practical usefulness and correct categorization. Skills rated below practical threshold have been removed.

## Categories

| Category | Skills | Avg Rating | What it covers |
|----------|--------|------------|----------------|
| [Evaluation & Benchmarking](#evaluation-benchmarking) | 92 | 4.9/10 ★★ | Benchmarks, metrics, scoring, model assessment |
| [Security & Safety](#security-safety) | 80 | 5.9/10 ★★★ | Jailbreaks, adversarial attacks, guardrails, prompt injection, red teaming |
| [Domain-Specific](#domain-specific) | 60 | 5.1/10 ★★★ | Medical, legal, financial, clinical, robotics applications |
| [Agentic Systems](#agentic-systems) | 54 | 5.7/10 ★★★ | Autonomous agents, tool use, planning, task decomposition |
| [Code & Software Engineering](#code-software-engineering) | 46 | 6.6/10 ★★★ | Code generation, bug detection, testing, refactoring, repair |
| [Reasoning & Chain-of-Thought](#reasoning-chain-of-thought) | 46 | 5.7/10 ★★★ | Chain-of-thought, logical inference, step-by-step reasoning, math |
| [Multi-Agent Systems](#multi-agent-systems) | 45 | 5.3/10 ★★★ | Agent collaboration, swarms, orchestration, debate frameworks |
| [RAG & Retrieval](#rag-retrieval) | 42 | 5.6/10 ★★★ | Retrieval-augmented generation, search, reranking, chunking |
| [Efficiency & Optimization](#efficiency-optimization) | 34 | 5.4/10 ★★★ | Quantization, pruning, compression, acceleration |
| [Data Processing](#data-processing) | 28 | 6.0/10 ★★★ | ETL, parsing, extraction, annotation, pipelines |
| [Prompt Engineering](#prompt-engineering) | 27 | 5.8/10 ★★★ | In-context learning, few-shot, instruction design, prompt optimization |
| [NLP & Text](#nlp-text) | 24 | 4.9/10 ★★ | Classification, summarization, translation, NER, QA |
| [Multimodal](#multimodal) | 21 | 5.0/10 ★★★ | Vision-language, audio, video, speech, cross-modal reasoning |
| [Memory & Context](#memory-context) | 20 | 5.6/10 ★★★ | Long-context handling, KV cache optimization, context compression |
| [Knowledge Graphs](#knowledge-graphs) | 18 | 5.2/10 ★★★ | Graph-based knowledge, ontologies, entity relations |
| [Explainability](#explainability) | 7 | 5.0/10 ★★ | Interpretability, attribution, transparency, causal analysis |
| [Fine-tuning & Training](#fine-tuning-training) | 7 | 4.3/10 ★★ | RLHF, GRPO, distillation, curriculum learning, reward models |
| **Total** | **651** | **5.5/10** | |

---

## Evaluation & Benchmarking

**92 skills** | Avg rating: 4.9/10

| Skill | Description | Rating | Lines |
|-------|-------------|--------|-------|
| [automated-structural-testing-llm-based](skills/automated-structural-testing-llm-based/SKILL.md) | Write structural tests for LLM-based agents using trace-based assertions, mocked LLM responses, and the test automati... | 8/10 ★★★★ | 242 |
| [testexplora-benchmarking-proactive-bug](skills/testexplora-benchmarking-proactive-bug/SKILL.md) | Proactive bug discovery through documentation-driven test generation. Generates tests that find latent bugs by compar... | 7/10 ★★★★ | 218 |
| [when-better-prompts-hurt](skills/when-better-prompts-hurt/SKILL.md) | Evaluation-driven prompt iteration using the Define-Test-Diagnose-Fix loop and Minimum Viable Evaluation Suite (MVES)... | 7/10 ★★★★ | 196 |
| [agent-based-software-artifact-evaluation](skills/agent-based-software-artifact-evaluation/SKILL.md) | Automatically evaluate software research artifacts (code repositories with READMEs) by constructing dependency-aware ... | 6/10 ★★★ | 203 |
| [aidev-studying-ai-coding](skills/aidev-studying-ai-coding/SKILL.md) | Analyze AI coding agent activity on GitHub repositories using the AIDev methodology. Identify agentic PRs, measure ag... | 6/10 ★★★ | 195 |
| [automated-multiple-mini-interview](skills/automated-multiple-mini-interview/SKILL.md) | Multi-agent framework for scoring subjective, open-ended responses (interviews, essays, reflections) using transcript... | 6/10 ★★★ | 189 |
| [biases-blind-spot-detecting](skills/biases-blind-spot-detecting/SKILL.md) | Automated black-box pipeline for detecting unverbalized biases in LLM decision-making. Discovers biases that models e... | 6/10 ★★★ | 194 |
| [cvedrl-code-verifier-difficulty-aware](skills/cvedrl-code-verifier-difficulty-aware/SKILL.md) | Generate difficulty-aware unit tests that verify LLM-generated code using branch coverage analysis, complexity-weight... | 6/10 ★★★ | 195 |
| [generating-data-driven-reasoning-rubrics](skills/generating-data-driven-reasoning-rubrics/SKILL.md) | Build granular error taxonomies from incorrect reasoning traces, then use those rubrics to detect errors in LLM outpu... | 6/10 ★★★ | 170 |
| [jaf-judge-agent-forest](skills/jaf-judge-agent-forest/SKILL.md) | Implement the Judge Agent Forest (JAF) pattern: evaluate and refine AI-generated outputs by judging cohorts of relate... | 6/10 ★★★ | 213 |
| [noisy-but-valid-robust](skills/noisy-but-valid-robust/SKILL.md) | Statistically certify LLM safety/quality using imperfect LLM judges with guaranteed Type-I error control. Implements ... | 6/10 ★★★ | 242 |
| [omnicode-benchmark-evaluating-software](skills/omnicode-benchmark-evaluating-software/SKILL.md) | Evaluate and improve code across four software engineering dimensions: bug fixing, test generation, code review fixin... | 6/10 ★★★ | 199 |
| [on-use-generate-dataset](skills/on-use-generate-dataset/SKILL.md) | Generate diverse, validated datasets of neural network implementations using LLM-driven combinatorial design. Use whe... | 6/10 ★★★ | 195 |
| [precise-reducing-bias-evaluations](skills/precise-reducing-bias-evaluations/SKILL.md) | Implement the PRECISE framework to debias LLM-as-judge evaluations of search, ranking, and RAG systems by combining a... | 6/10 ★★★ | 212 |
| [rubberduckbench-benchmark-ai-coding](skills/rubberduckbench-benchmark-ai-coding/SKILL.md) | Evaluate and improve AI coding assistant responses using RubberDuckBench's rubric-based methodology. Detects hallucin... | 6/10 ★★★ | 182 |
| [when-get-significantly-worse](skills/when-get-significantly-worse/SKILL.md) | Statistically detect LLM degradation after optimization using McNemar's paired test. Use when: 'did quantization hurt... | 6/10 ★★★ | 245 |
| [agentdrive-open-benchmark-dataset](skills/agentdrive-open-benchmark-dataset/SKILL.md) | Generate structured autonomous driving scenarios and MCQ benchmarks using AgentDrive's factorized 7-axis prompt-to-JS... | 5/10 ★★ | 284 |
| [automated-rubrics-reliable-evaluation](skills/automated-rubrics-reliable-evaluation/SKILL.md) | Generate fine-grained evaluation rubrics for medical dialogue systems using a retrieval-augmented multi-agent pipelin... | 5/10 ★★ | 180 |
| [benchmarking-reward-hack-detection](skills/benchmarking-reward-hack-detection/SKILL.md) | Detect reward hacking in AI-generated code trajectories using contrastive analysis from the TRACE benchmark. Use when... | 5/10 ★★ | 190 |
| [benchmarking-uncertainty-calibration-long-form](skills/benchmarking-uncertainty-calibration-long-form/SKILL.md) | Implement uncertainty quantification and calibration assessment for LLM-generated long-form answers. Apply answer-fre... | 5/10 ★★ | 210 |
| [beyond-needles-illusion-decoupled](skills/beyond-needles-illusion-decoupled/SKILL.md) | Decouple evidence access from evidence use when evaluating or building long-context and RAG systems under semantic in... | 5/10 ★★ | 191 |
| [biasscope-automated-detection-bias](skills/biasscope-automated-detection-bias/SKILL.md) | Automatically discover and test for hidden biases in LLM-as-a-Judge evaluation pipelines using the BiasScope framewor... | 5/10 ★★ | 206 |
| [can-reasoning-be-trusted](skills/can-reasoning-be-trusted/SKILL.md) | Validate and score LLM-generated statistical reasoning using a three-axis rubric (Correctness 40%, Explanation 35%, R... | 5/10 ★★ | 203 |
| [capture-flags-family-based-evaluation](skills/capture-flags-family-based-evaluation/SKILL.md) | Generate semantics-preserving variants of Python CTF challenges to stress-test agentic LLM robustness. Applies the Ev... | 5/10 ★★ | 165 |
| [compar-ia-french-governments](skills/compar-ia-french-governments/SKILL.md) | Build multilingual LLM evaluation arenas and preference data collection pipelines modeled on France's compar:IA platf... | 5/10 ★★ | 310 |
| [comparing-ai-coding-agents](skills/comparing-ai-coding-agents/SKILL.md) | Analyze AI coding agent PR datasets using task-stratified acceptance rate methodology. Classify PRs into 9 task categ... | 5/10 ★★ | 207 |
| [compass-contrastive-learning-automated](skills/compass-contrastive-learning-automated/SKILL.md) | Assess patch correctness using contrastive learning on code representations. Applies semantic-preserving code transfo... | 5/10 ★★ | 180 |
| [completing-missing-annotation-multi-agent](skills/completing-missing-annotation-multi-agent/SKILL.md) | Multi-agent debate framework for relevance assessment and annotation completion. Uses opposing-stance LLM agents with... | 5/10 ★★ | 251 |
| [comprehensive-evaluation-software-engineering](skills/comprehensive-evaluation-software-engineering/SKILL.md) | Evaluate and optimize LLM-driven software engineering workflows across five task types (bug fixing, feature developme... | 5/10 ★★ | 285 |
| [cross-lingual-stability-judges-under](skills/cross-lingual-stability-judges-under/SKILL.md) | Detect and fix cross-lingual evaluation instabilities in LLM-as-a-judge pipelines. Use when: 'audit my multilingual e... | 5/10 ★★ | 199 |
| [dial-summer-structured-evaluation-framework](skills/dial-summer-structured-evaluation-framework/SKILL.md) | Evaluate dialogue summaries using the DIAL-SUMMER hierarchical error taxonomy. Detects 10 fine-grained error types ac... | 5/10 ★★ | 244 |
| [echoes-loop-diagnosing-risks](skills/echoes-loop-diagnosing-risks/SKILL.md) | Diagnose and mitigate feedback-loop risks (bias amplification, hallucination propagation, exposure polarization) in L... | 5/10 ★★ | 395 |
| [entworld-holistic-environment-benchmark](skills/entworld-holistic-environment-benchmark/SKILL.md) | Build verifiable enterprise GUI agent benchmarks using schema-grounded task generation and SQL-based deterministic ve... | 5/10 ★★ | 158 |
| [es-memeval-benchmarking-conversational-agents](skills/es-memeval-benchmarking-conversational-agents/SKILL.md) | Build and evaluate long-term memory systems for conversational agents using the ES-MemEval five-capability framework ... | 5/10 ★★ | 226 |
| [evaluating-they-not-know](skills/evaluating-they-not-know/SKILL.md) | Build statistically efficient LLM evaluation pipelines that combine direct accuracy with pairwise comparison signals ... | 5/10 ★★ | 185 |
| [evaluation-entity-matching-recommender](skills/evaluation-entity-matching-recommender/SKILL.md) | Build and evaluate cross-dataset entity matching pipelines for recommender systems. Implements the Reddit-Amazon-EM m... | 5/10 ★★ | 182 |
| [evaluation-legal-applications-challenges](skills/evaluation-legal-applications-challenges/SKILL.md) | Build evaluation pipelines for LLMs in legal tasks using a three-dimensional framework: outcome correctness, reasonin... | 5/10 ★★ | 171 |
| [evermembench-benchmarking-long-term-interactive](skills/evermembench-benchmarking-long-term-interactive/SKILL.md) | Build and evaluate long-term conversational memory systems for multi-party, multi-topic dialogues. Implements the Eve... | 5/10 ★★ | 182 |
| [exploring-reasoning-reward-agents](skills/exploring-reasoning-reward-agents/SKILL.md) | Apply Agent Reasoning Reward Model (Agent-RRM) structured critique to improve multi-step agent trajectories. Evaluate... | 5/10 ★★ | 219 |
| [featurebench-benchmarking-agentic-coding](skills/featurebench-benchmarking-agentic-coding/SKILL.md) | Extract feature-level coding tasks from repositories using test-driven dependency graph tracing. Use when the user sa... | 5/10 ★★ | 178 |
| [gender-race-bias-consumer](skills/gender-race-bias-consumer/SKILL.md) | Audit LLM-generated product recommendations for gender and race bias using marked words analysis, SVM classification,... | 5/10 ★★ | 255 |
| [helm-human-centered-evaluation-framework](skills/helm-human-centered-evaluation-framework/SKILL.md) | Evaluate LLM-powered recommender systems across five human-centered dimensions: Intent Alignment, Explanation Quality... | 5/10 ★★ | 244 |
| [how-much-reasoning-retrieval-augmented](skills/how-much-reasoning-retrieval-augmented/SKILL.md) | Build contamination-aware hybrid RAG evaluation pipelines that couple knowledge graphs with text retrieval for multi-... | 5/10 ★★ | 178 |
| [icon-intent-context-coupling-multi-turn](skills/icon-intent-context-coupling-multi-turn/SKILL.md) | Build multi-turn LLM safety evaluation harnesses using the Intent-Context Coupling framework from ICON. Generates str... | 5/10 ★★ | 243 |
| [linglanmidian-systematic-evaluation-tcm](skills/linglanmidian-systematic-evaluation-tcm/SKILL.md) | Build rigorous, multi-task evaluation benchmarks for domain-specific LLMs using the LingLanMiDian methodology: synony... | 5/10 ★★ | 245 |
| [lingxidiagbench-multi-agent-framework-benchmarking](skills/lingxidiagbench-multi-agent-framework-benchmarking/SKILL.md) | Build multi-agent benchmarking systems with role-separated agents (simulator, interviewer, evaluator) for structured ... | 5/10 ★★ | 216 |
| [livemedbench-contamination-free-medical-benchmark](skills/livemedbench-contamination-free-medical-benchmark/SKILL.md) | Build contamination-free LLM evaluation pipelines with multi-agent data curation and automated rubric-based scoring. ... | 5/10 ★★ | 296 |
| [logicscore-fine-grained-logic-evaluation](skills/logicscore-fine-grained-logic-evaluation/SKILL.md) | Evaluate the logical integrity of LLM-generated multi-hop answers using Horn Rule backward chaining. Scores Completen... | 5/10 ★★ | 185 |
| [made-benchmark-environments-closed-loop](skills/made-benchmark-environments-closed-loop/SKILL.md) | Build closed-loop discovery benchmarks where an agent iteratively proposes, evaluates, and refines candidates under a... | 5/10 ★★ | 144 |
| [masalbench-benchmark-contextual-cross-cultural](skills/masalbench-benchmark-contextual-cross-cultural/SKILL.md) | Build cross-cultural figurative language benchmarks and evaluation pipelines for LLMs. Applies the MasalBench methodo... | 5/10 ★★ | 186 |
| [mcp-atlas-large-scale-benchmark-tool-use](skills/mcp-atlas-large-scale-benchmark-tool-use/SKILL.md) | Design and evaluate multi-server MCP tool-use benchmarks using claims-based scoring rubrics. Use when: 'benchmark my ... | 5/10 ★★ | 210 |
| [mhdash-online-platform-benchmarking](skills/mhdash-online-platform-benchmarking/SKILL.md) | Build risk-aware evaluation pipelines for mental health AI assistants using the MHDash framework. Implements multi-di... | 5/10 ★★ | 285 |
| [mpib-benchmark-medical-prompt](skills/mpib-benchmark-medical-prompt/SKILL.md) | Evaluate and defend clinical LLM systems against prompt injection attacks using the MPIB benchmark methodology. Imple... | 5/10 ★★ | 177 |
| [omni-rrm-advancing-omni-reward](skills/omni-rrm-advancing-omni-reward/SKILL.md) | Build rubric-grounded reward models and preference evaluation pipelines for multimodal AI outputs. Use when asked to ... | 5/10 ★★ | 180 |
| [robustexplain-evaluating-robustness-llm-based](skills/robustexplain-evaluating-robustness-llm-based/SKILL.md) | Evaluate robustness of LLM-generated recommendation explanations under realistic user behavior noise. Use when: 'test... | 5/10 ★★ | 211 |
| [seta-statistical-fault-attribution](skills/seta-statistical-fault-attribution/SKILL.md) | Diagnose and attribute faults in compound AI systems (multi-model pipelines) using SETA's modular robustness testing ... | 5/10 ★★ | 247 |
| [sparseeval-evaluation-sparse-optimization](skills/sparseeval-evaluation-sparse-optimization/SKILL.md) | Efficiently evaluate LLMs on benchmarks by selecting a small subset of anchor items via sparse optimization, reproduc... | 5/10 ★★ | 221 |
| [sycoeval-em-sycophancy-evaluation-simulated](skills/sycoeval-em-sycophancy-evaluation-simulated/SKILL.md) | Build multi-agent adversarial simulations to evaluate LLM sycophancy and policy compliance under social pressure. Use... | 5/10 ★★ | 241 |
| [taming-scylla-understanding-multi-headed](skills/taming-scylla-understanding-multi-headed/SKILL.md) | Evaluate and optimize agentic coding tool configurations using Scylla's tiered ablation framework and Cost-of-Pass (C... | 5/10 ★★ | 165 |
| [the-necessity-unified-framework](skills/the-necessity-unified-framework/SKILL.md) | Design and implement standardized, reproducible evaluation harnesses for LLM-based agents. Eliminates confounding fac... | 5/10 ★★ | 185 |
| [uncertainty-and-fairness-awareness](skills/uncertainty-and-fairness-awareness/SKILL.md) | Audit LLM-based recommendation systems for predictive uncertainty and demographic fairness bias. Implements the SNSR/... | 5/10 ★★ | 230 |
| [understanding-dominant-themes-reviewing](skills/understanding-dominant-themes-reviewing/SKILL.md) | Analyze code review comments on AI-authored PRs to identify dominant review themes using a 12-category taxonomy deriv... | 5/10 ★★ | 226 |
| [vectra-metric-dataset-visual](skills/vectra-metric-dataset-visual/SKILL.md) | Assess visual quality of translated product images using Vectra's 14-dimension scoring framework. Use when: 'evaluate... | 5/10 ★★ | 305 |
| [who-gets-which-message](skills/who-gets-which-message/SKILL.md) | Audit demographic bias in LLM-generated targeted text. Detects age- and gender-based stereotyping in personalized mes... | 5/10 ★★ | 221 |
| [why-deep-research-agent](skills/why-deep-research-agent/SKILL.md) | Audit and diagnose hallucinations in multi-step AI research agent workflows using the PIES taxonomy (Planning/Summari... | 5/10 ★★ | 199 |
| [wiki-live-challenge-challenging](skills/wiki-live-challenge-challenging/SKILL.md) | Evaluate deep research agents and LLM-generated long-form articles using the Wiki Live Challenge framework: 39 fine-g... | 5/10 ★★ | 264 |
| [3-secbench-large-scale-evaluation-suite-security](skills/3-secbench-large-scale-evaluation-suite-security/SKILL.md) | Evaluate and harden LLM-based autonomous agents against adversarial attacks using the α³-SecBench layered security fr... | 4/10 ★★ | 182 |
| [apex-agents](skills/apex-agents/SKILL.md) | Design and execute long-horizon, cross-application agent workflows for professional knowledge work (finance, consulti... | 4/10 ★★ | 223 |
| [assessing-quality-mental-health](skills/assessing-quality-mental-health/SKILL.md) | Evaluate LLM-generated mental health responses using a 6-attribute clinical rubric spanning Cognitive Support (Guidan... | 4/10 ★★ | 374 |
| [bass-benchmarking-audio-lms](skills/bass-benchmarking-audio-lms/SKILL.md) | Build evaluation benchmarks for audio language models using the BASS methodology — structured task taxonomies across ... | 4/10 ★★ | 260 |
| [bioace-automated-framework-biomedical](skills/bioace-automated-framework-biomedical/SKILL.md) | Evaluate biomedical QA outputs using the BioACE nugget-based framework — assess answer completeness, correctness, pre... | 4/10 ★★ | 238 |
| [creditaudit-2textnd-dimension-evaluation](skills/creditaudit-2textnd-dimension-evaluation/SKILL.md) | Evaluate and select LLMs using CreditAudit's 2D framework: mean ability plus stability risk (fluctuation) across syst... | 4/10 ★★ | 211 |
| [decomposing-reasoning-efficiency](skills/decomposing-reasoning-efficiency/SKILL.md) | Analyze and optimize LLM reasoning token efficiency using a multiplicative decomposition framework. Breaks down reaso... | 4/10 ★★ | 248 |
| [do-vlms-have-moral](skills/do-vlms-have-moral/SKILL.md) | Audit and harden the moral robustness of Vision-Language Model (VLM) pipelines against adversarial perturbations that... | 4/10 ★★ | 211 |
| [evaluating-social-bias-rag](skills/evaluating-social-bias-rag/SKILL.md) | Evaluate and mitigate social bias in RAG pipelines. Use when: 'audit my RAG system for bias', 'check if retrieval int... | 4/10 ★★ | 212 |
| [from-helpfulness-toxic-proactivity](skills/from-helpfulness-toxic-proactivity/SKILL.md) | Diagnose and mitigate Toxic Proactivity in LLM agent systems -- the failure mode where agents override ethical constr... | 4/10 ★★ | 194 |
| [halluverse-m3-multitask-multilingual-benchmark-hal](skills/halluverse-m3-multitask-multilingual-benchmark-hal/SKILL.md) | Detect and classify hallucinations in LLM outputs across languages using the HalluVerse-M3 fine-grained taxonomy (ent... | 4/10 ★★ | 148 |
| [how-well-open-sourced](skills/how-well-open-sourced/SKILL.md) | Select and deploy AI-generated image detection models based on threat-landscape analysis using zero-shot benchmark da... | 4/10 ★★ | 232 |
| [humans-welcome-observe-first-look](skills/humans-welcome-observe-first-look/SKILL.md) | Analyze AI agent social network activity using topic taxonomy classification and multi-level toxicity scoring. Detect... | 4/10 ★★ | 191 |
| [isd-agent-bench-comprehensive-benchmark-evaluating](skills/isd-agent-bench-comprehensive-benchmark-evaluating/SKILL.md) | Build and evaluate LLM-based Instructional Design agents using the ADDIE framework, Context Matrix scenario generatio... | 4/10 ★★ | 210 |
| [jobresqa-benchmark-machine-reading](skills/jobresqa-benchmark-machine-reading/SKILL.md) | Build and evaluate multilingual machine reading comprehension systems for HR documents (resumes and job descriptions)... | 4/10 ★★ | 152 |
| [livibench-omnimodal-benchmark-interactive](skills/livibench-omnimodal-benchmark-interactive/SKILL.md) | Build omnimodal benchmarks and evaluation pipelines for interactive video understanding (livestreams, real-time comme... | 4/10 ★★ | 238 |
| [odysseyarena-benchmarking-long-horizon-active](skills/odysseyarena-benchmarking-long-horizon-active/SKILL.md) | Design and run inductive agent benchmarks where LLMs must discover hidden rules through long-horizon interaction loop... | 4/10 ★★ | 176 |
| [parse-open-domain-reasoning-question](skills/parse-open-domain-reasoning-question/SKILL.md) | Build and evaluate reasoning-focused QA systems for low-resource languages using the PARSE methodology: structured pr... | 4/10 ★★ | 220 |
| [predictive-coding-information-bottleneck](skills/predictive-coding-information-bottleneck/SKILL.md) | Build lightweight hallucination detection pipelines using Predictive Coding surprise signals and Information Bottlene... | 4/10 ★★ | 238 |
| [proxywar-dynamic-assessment-of](skills/proxywar-dynamic-assessment-of/SKILL.md) | Build competitive game-arena evaluation frameworks for LLM-generated code using ProxyWar's multi-layer pipeline: agen... | 4/10 ★★ | 197 |
| [socialveil-probing-social-intelligence](skills/socialveil-probing-social-intelligence/SKILL.md) | Stress-test LLM agents' social intelligence by injecting realistic communication barriers (semantic vagueness, socioc... | 4/10 ★★ | 203 |
| [steereval-framework-evaluating-steerability](skills/steereval-framework-evaluating-steerability/SKILL.md) | Evaluate and improve the steerability of natural-language-profile-based recommender systems using the SteerEval frame... | 4/10 ★★ | 191 |
| [the-clef-2026-finmmeval-lab](skills/the-clef-2026-finmmeval-lab/SKILL.md) | Build multilingual, multimodal financial AI evaluation pipelines using the FinMMEval framework. Covers financial exam... | 4/10 ★★ | 246 |
| [trapped-past-disentangling-fluid](skills/trapped-past-disentangling-fluid/SKILL.md) | Diagnose whether an LLM is memorizing or reasoning by constructing distributional proximity tests. Classifies task in... | 4/10 ★★ | 174 |
| [tsrbench-comprehensive-multi-task-multi-modal](skills/tsrbench-comprehensive-multi-task-multi-modal/SKILL.md) | Evaluate and build multi-modal time series reasoning pipelines using the TSRBench framework. Covers perception, reaso... | 4/10 ★★ | 206 |
| [beyond-instrumental-substitutive-paradigms](skills/beyond-instrumental-substitutive-paradigms/SKILL.md) | Audit and diagnose cultural bias artifacts in LLM-powered applications using the Machine Culture framework. Detects C... | 3/10 ★★ | 189 |

---

## Security & Safety

**80 skills** | Avg rating: 5.9/10

| Skill | Description | Rating | Lines |
|-------|-------------|--------|-------|
| [agenticscr-an-autonomous-agentic](skills/agenticscr-an-autonomous-agentic/SKILL.md) | Agentic secure code review for detecting immature vulnerabilities at pre-commit stage. Uses a two-phase Detector-Vali... | 8/10 ★★★★ | 177 |
| [beyond-function-level-analysis-context-aware](skills/beyond-function-level-analysis-context-aware/SKILL.md) | Inter-procedural vulnerability detection using context-aware reasoning. Analyzes functions alongside their callers, c... | 8/10 ★★★★ | 170 |
| [issueguard-real-time-secret-leak](skills/issueguard-real-time-secret-leak/SKILL.md) | Scan text for leaked secrets using a two-stage pipeline: regex candidate extraction followed by contextual classifica... | 8/10 ★★★★ | 268 |
| [prompt-injection-attacks-agentic](skills/prompt-injection-attacks-agentic/SKILL.md) | Analyze, detect, and defend against prompt injection attacks targeting agentic coding assistants (Claude Code, Copilo... | 8/10 ★★★★ | 207 |
| [rvb-automating-ai-system](skills/rvb-automating-ai-system/SKILL.md) | Harden code and AI guardrails through iterative Red Team vs Blue Team adversarial games. Use when the user says 'hard... | 8/10 ★★★★ | 222 |
| [sifting-noise-comparative-study](skills/sifting-noise-comparative-study/SKILL.md) | Filter false positives from static analysis security tools (SAST) using LLM-agent-driven triage. Applies iterative co... | 8/10 ★★★★ | 154 |
| [agent-fence-mapping-security-vulnerabilities](skills/agent-fence-mapping-security-vulnerabilities/SKILL.md) | Audit LLM agent systems for trust-boundary security vulnerabilities using the AgentFence taxonomy of 14 attack classe... | 7/10 ★★★★ | 211 |
| [agentsys-secure-dynamic-agents](skills/agentsys-secure-dynamic-agents/SKILL.md) | Build LLM agent systems hardened against indirect prompt injection using hierarchical memory isolation, schema-valida... | 7/10 ★★★★ | 191 |
| [an-cost-efficient-agentic-framework](skills/an-cost-efficient-agentic-framework/SKILL.md) | Audit Ethereum smart contracts for business logic vulnerabilities using Heimdallr's four-phase agentic pipeline: func... | 7/10 ★★★★ | 202 |
| [breaking-protocol-security-analysis](skills/breaking-protocol-security-analysis/SKILL.md) | Audit and harden Model Context Protocol (MCP) server deployments against protocol-level vulnerabilities including cap... | 7/10 ★★★★ | 264 |
| [co-redteam-orchestrated-security-discovery](skills/co-redteam-orchestrated-security-discovery/SKILL.md) | Multi-agent security vulnerability discovery and exploitation using Co-RedTeam's orchestrated workflow. Decomposes se... | 7/10 ★★★★ | 197 |
| [constitutional-spec-driven-development-enforcing](skills/constitutional-spec-driven-development-enforcing/SKILL.md) | Enforce security by construction in AI-generated code using Constitutional Spec-Driven Development (CSDD). Creates a ... | 7/10 ★★★★ | 258 |
| [cutting-gordian-knot-detecting](skills/cutting-gordian-knot-detecting/SKILL.md) | Detect malicious PyPI/NPM packages using behavioral pattern mining and semantic reasoning (PyGuard). Use when: 'scan ... | 7/10 ★★★★ | 212 |
| [evaluating-enhancing-vulnerability-reasoning](skills/evaluating-enhancing-vulnerability-reasoning/SKILL.md) | Perform DAG-structured vulnerability reasoning on code, modeling causal dependencies between code facts instead of li... | 7/10 ★★★★ | 211 |
| [from-assistant-double-agent](skills/from-assistant-double-agent/SKILL.md) | Security audit and hardening for personalized LLM-based agents against prompt injection, tool poisoning, and memory a... | 7/10 ★★★★ | 230 |
| [from-detection-prevention-explaining](skills/from-detection-prevention-explaining/SKILL.md) | Proactively identify security-critical code regions and generate prevention-oriented explanations before vulnerabilit... | 7/10 ★★★★ | 265 |
| [malicious-agent-skills-wild](skills/malicious-agent-skills-wild/SKILL.md) | Audit and detect malicious agent skills in LLM skill registries using the taxonomy and analysis pipeline from Liu et ... | 7/10 ★★★★ | 264 |
| [seccodeprm-process-reward-code](skills/seccodeprm-process-reward-code/SKILL.md) | Step-level security scoring for code generation and vulnerability detection using process reward model techniques. Us... | 7/10 ★★★★ | 201 |
| [secure-code-generation-via](skills/secure-code-generation-via/SKILL.md) | Generates secure, vulnerability-free code by applying the SecCoderX reasoning framework — systematically analyzing co... | 7/10 ★★★★ | 254 |
| [the-landscape-prompt-injection](skills/the-landscape-prompt-injection/SKILL.md) | Harden LLM agent systems against prompt injection using layered text/model/execution defenses and the AgentPI evaluat... | 7/10 ★★★★ | 244 |
| [the-semantic-trap-fine-tuned](skills/the-semantic-trap-fine-tuned/SKILL.md) | Evaluate code vulnerability detection for semantic traps -- where analysis fixates on functional context (e.g., "this... | 7/10 ★★★★ | 179 |
| [vulread-knowledge-graph-guided-software-vulnerabil](skills/vulread-knowledge-graph-guided-software-vulnerabil/SKILL.md) | CWE-guided vulnerability reasoning and detection using knowledge-graph-structured analysis. Analyzes source code for ... | 7/10 ★★★★ | 212 |
| [yasa-scalable-multi-language-taint](skills/yasa-scalable-multi-language-taint/SKILL.md) | Perform unified multi-language taint analysis across Java, JavaScript, Python, and Go codebases using YASA's UAST-bas... | 7/10 ★★★★ | 204 |
| [aegis-governance-integrity-security](skills/aegis-governance-integrity-security/SKILL.md) | Red-team and harden AI voice agents and LLM-powered service systems against adversarial misuse using the Aegis framew... | 6/10 ★★★ | 237 |
| [agent2agent-threats-safety-critical-assistants](skills/agent2agent-threats-safety-critical-assistants/SKILL.md) | Threat model multi-agent LLM systems using the AgentHeLLM framework -- formally separating asset identification from ... | 6/10 ★★★ | 207 |
| [covagent-overcoming-30-curse](skills/covagent-overcoming-30-curse/SKILL.md) | Boost Android app test coverage beyond the 30% activity ceiling using agentic static analysis of Smali code, componen... | 6/10 ★★★ | 174 |
| [cve-factory-scaling-expert-level-agentic](skills/cve-factory-scaling-expert-level-agentic/SKILL.md) | Build multi-agent pipelines that transform CVE metadata into fully executable vulnerability reproduction environments... | 6/10 ★★★ | 240 |
| [eliciting-least-to-most-reasoning-phishing](skills/eliciting-least-to-most-reasoning-phishing/SKILL.md) | Detect phishing URLs using Least-to-Most iterative decomposition with answer sensitivity scoring. Triggers: 'analyze ... | 6/10 ★★★ | 155 |
| [extracting-recurring-vulnerabilities-black-box](skills/extracting-recurring-vulnerabilities-black-box/SKILL.md) | Predict and prevent recurring vulnerabilities in LLM-generated code using the FSTab (Feature-Security Table) techniqu... | 6/10 ★★★ | 188 |
| [following-dragons-code-review-guided](skills/following-dragons-code-review-guided/SKILL.md) | Extract security-relevant signals from code review comments and translate them into fuzzer-guiding annotations using ... | 6/10 ★★★ | 158 |
| [helios-hierarchical-graph-abstraction](skills/helios-hierarchical-graph-abstraction/SKILL.md) | Structure-aware binary decompilation using hierarchical control-flow graph abstraction for LLMs. Converts binary prog... | 6/10 ★★★ | 204 |
| [hidden-licensing-risks-llmware](skills/hidden-licensing-risks-llmware/SKILL.md) | Detect license incompatibilities across LLM supply chains (OSS repos, models, datasets) using the LiAgent multi-agent... | 6/10 ★★★ | 182 |
| [icl-evader-zero-query-black-box-evasion](skills/icl-evader-zero-query-black-box-evasion/SKILL.md) | Harden ICL classification prompts against zero-query black-box evasion attacks. Audit in-context learning pipelines f... | 6/10 ★★★ | 251 |
| [identifying-adversary-tactics-techniques](skills/identifying-adversary-tactics-techniques/SKILL.md) | Identify MITRE ATT&CK Tactics, Techniques, and Procedures (TTPs) in decompiled malware binaries using the TTPDetect m... | 6/10 ★★★ | 198 |
| [multi-agent-end-to-end-vulnerability-management](skills/multi-agent-end-to-end-vulnerability-management/SKILL.md) | Detect, confirm, repair, and validate recurring software vulnerabilities using a multi-agent pipeline modeled on MAVM... | 6/10 ★★★ | 196 |
| [mulvul-retrieval-augmented-multi-agent-code](skills/mulvul-retrieval-augmented-multi-agent-code/SKILL.md) | Multi-agent vulnerability detection using coarse-to-fine routing, contrastive retrieval, and cross-model prompt evolu... | 6/10 ★★★ | 204 |
| [naamse-framework-evolutionary-security](skills/naamse-framework-evolutionary-security/SKILL.md) | Implement evolutionary security evaluation for AI agents using the NAAMSE framework — genetic prompt mutation, hierar... | 6/10 ★★★ | 201 |
| [next-gen-captchas-leveraging-cognitive](skills/next-gen-captchas-leveraging-cognitive/SKILL.md) | Design and implement AI-resistant CAPTCHA systems that exploit the cognitive gap between humans and GUI agents. Cover... | 6/10 ★★★ | 253 |
| [realsec-bench-benchmark-evaluating-secure](skills/realsec-bench-benchmark-evaluating-secure/SKILL.md) | Evaluate and improve secure code generation using the RealSec-bench methodology: multi-stage vulnerability detection ... | 6/10 ★★★ | 195 |
| [redsage-cybersecurity-generalist](skills/redsage-cybersecurity-generalist/SKILL.md) | Apply RedSage's agentic augmentation methodology to cybersecurity assistance: structured threat analysis, vulnerabili... | 6/10 ★★★ | 234 |
| [safepred-predictive-guardrail-computer-using](skills/safepred-predictive-guardrail-computer-using/SKILL.md) | Implement predictive safety guardrails for computer-using agents and automated pipelines using world-model-based risk... | 6/10 ★★★ | 179 |
| [spider-sense-intrinsic-risk-sensing](skills/spider-sense-intrinsic-risk-sensing/SKILL.md) | Implement event-driven, hierarchical security screening for LLM agent systems using Intrinsic Risk Sensing. Adds late... | 6/10 ★★★ | 212 |
| [stateless-yet-not-forgetful](skills/stateless-yet-not-forgetful/SKILL.md) | Detect, audit, and defend against implicit memory channels in LLM-powered systems where models encode hidden state in... | 6/10 ★★★ | 245 |
| [stepshield-not-whether-intervene](skills/stepshield-not-whether-intervene/SKILL.md) | Implement temporal safety monitoring for AI agent trajectories using StepShield's cascaded HybridGuard pattern. Detec... | 6/10 ★★★ | 282 |
| [the-compliance-paradox-semantic-instruction](skills/the-compliance-paradox-semantic-instruction/SKILL.md) | Detect and defend against adversarial prompt injections hidden in code submissions that exploit LLM instruction-follo... | 6/10 ★★★ | 229 |
| [whispers-wealth-red-teaming-googles](skills/whispers-wealth-red-teaming-googles/SKILL.md) | Red-team LLM-based agentic payment systems against prompt injection attacks targeting transaction integrity and crede... | 6/10 ★★★ | 218 |
| [agentdog-diagnostic-guardrail-framework](skills/agentdog-diagnostic-guardrail-framework/SKILL.md) | Implement diagnostic safety guardrails for AI agent systems using the AgentDoG three-dimensional taxonomy (risk sourc... | 5/10 ★★ | 340 |
| [anonymization-enhanced-privacy-protection-mobile-g](skills/anonymization-enhanced-privacy-protection-mobile-g/SKILL.md) | Implement available-but-invisible privacy protection for mobile GUI agents using PII-aware anonymization with determi... | 5/10 ★★ | 178 |
| [autoregressive-yet-revisable-decoding-revision](skills/autoregressive-yet-revisable-decoding-revision/SKILL.md) | Generate secure code using Stream of Revision — an in-decoding self-correction technique that backtracks and patches ... | 5/10 ★★ | 201 |
| [benchmarking-zero-shot-few-shot-phishing](skills/benchmarking-zero-shot-few-shot-phishing/SKILL.md) | Detect phishing URLs using LLM zero-shot and few-shot prompting with structured classification prompts. Use when: 'cl... | 5/10 ★★ | 215 |
| [constructing-multi-label-hierarchical-classificati](skills/constructing-multi-label-hierarchical-classificati/SKILL.md) | Build multi-label hierarchical classifiers for MITRE ATT&CK text tagging using stage-wise classical ML (SGD-SVM + TF-... | 5/10 ★★ | 226 |
| [context-sensitive-pointer-analysis-arkts](skills/context-sensitive-pointer-analysis-arkts/SKILL.md) | Perform context-sensitive pointer analysis for ArkTS/TypeScript code targeting OpenHarmony. Build precise call graphs... | 5/10 ★★ | 223 |
| [david-vs-goliath-verifiable](skills/david-vs-goliath-verifiable/SKILL.md) | Audit and harden tool-augmented AI agent systems against Tag-Along Attacks -- adversarial agent-to-agent jailbreaks t... | 5/10 ★★ | 165 |
| [draincode-stealthy-energy-consumption](skills/draincode-stealthy-energy-consumption/SKILL.md) | Evaluate and defend RAG-based code generation systems against energy-drain attacks that poison retrieval contexts to ... | 5/10 ★★ | 221 |
| [fraudshield-knowledge-graph-empowered](skills/fraudshield-knowledge-graph-empowered/SKILL.md) | Detect and defend against fraudulent content in LLM inputs using knowledge-graph-augmented analysis. Builds a fraud t... | 5/10 ★★ | 269 |
| [gradingattack-attacking-short-answer](skills/gradingattack-attacking-short-answer/SKILL.md) | Audit LLM-based automatic short answer grading (ASAG) systems for adversarial vulnerabilities using token-level and p... | 5/10 ★★ | 243 |
| [hallucination-resistant-security-planning](skills/hallucination-resistant-security-planning/SKILL.md) | Generate reliable incident response and security recovery plans using a generate-check-refine loop with consistency-b... | 5/10 ★★ | 259 |
| [how-information-access-affect](skills/how-information-access-affect/SKILL.md) | Build Extract-and-Evaluate (EaE) hierarchical monitoring pipelines that detect sabotage and misbehavior in LLM agent ... | 5/10 ★★ | 194 |
| [jailbreaks-vision-multimodal-reasoning](skills/jailbreaks-vision-multimodal-reasoning/SKILL.md) | Defensive security skill for testing and hardening Vision-Language Models (VLMs) against multimodal jailbreak attacks... | 5/10 ★★ | 250 |
| [just-ask-curious-code](skills/just-ask-curious-code/SKILL.md) | Audit and defend LLM-powered applications against system prompt extraction attacks using the JustAsk framework's UCB-... | 5/10 ★★ | 224 |
| [llama-31-foundationai-securityllm-reasoning-8b-tec](skills/llama-31-foundationai-securityllm-reasoning-8b-tec/SKILL.md) | Apply Foundation-Sec-8B-Reasoning cybersecurity reasoning patterns: structured <think> chain-of-thought for CVE-to-CW... | 5/10 ★★ | 251 |
| [lps-bench-benchmarking-safety-awareness](skills/lps-bench-benchmarking-safety-awareness/SKILL.md) | Audit MCP-based agent workflows for planning-time safety risks using the LPS-Bench framework's 9 risk taxonomy (false... | 5/10 ★★ | 191 |
| [mempot-defending-against-memory](skills/mempot-defending-against-memory/SKILL.md) | Defend LLM agent memory systems against extraction attacks using optimized honeypot injection and sequential detectio... | 5/10 ★★ | 246 |
| [now-you-hear-me](skills/now-you-hear-me/SKILL.md) | Audit and defend large audio-language models (LALMs) against narrative-style audio jailbreaks. Based on the 'Now You ... | 5/10 ★★ | 311 |
| [patch-to-poc-systematic-study-agentic](skills/patch-to-poc-systematic-study-agentic/SKILL.md) | Agentic kernel vulnerability reproduction from security patches. Implements the K-Repro methodology: controlled code ... | 5/10 ★★ | 172 |
| [persona-jailbreaking](skills/persona-jailbreaking/SKILL.md) | Audit and defend LLM-powered applications against persona manipulation attacks using the PHISH framework (Persona Hij... | 5/10 ★★ | 301 |
| [physical-prompt-injection-attacks](skills/physical-prompt-injection-attacks/SKILL.md) | Defend against and red-team physical prompt injection attacks on Large Vision-Language Models (LVLMs). Build input sa... | 5/10 ★★ | 291 |
| [prism-xr-empowering-privacy-aware-xr](skills/prism-xr-empowering-privacy-aware-xr/SKILL.md) | Build privacy-aware pipelines that filter sensitive content from visual frames before sending to cloud AI models, usi... | 5/10 ★★ | 301 |
| [puda-private-user-dataset](skills/puda-private-user-dataset/SKILL.md) | Build privacy-preserving personalized AI systems using Puda's multi-granularity user data architecture. Implements cl... | 5/10 ★★ | 213 |
| [qrs-rule-synthesizing-neuro-symbolic-triad](skills/qrs-rule-synthesizing-neuro-symbolic-triad/SKILL.md) | Autonomous vulnerability discovery using the QRS (Query, Review, Sanitize) neuro-symbolic triad. Generates CodeQL que... | 5/10 ★★ | 229 |
| [query-efficient-agentic-graph-extraction](skills/query-efficient-agentic-graph-extraction/SKILL.md) | Implements the AGEA framework for budget-constrained extraction of knowledge graphs from GraphRAG systems using novel... | 5/10 ★★ | 239 |
| [reliable-responsible-foundation-comprehensive](skills/reliable-responsible-foundation-comprehensive/SKILL.md) | Audit and harden AI/ML systems for reliability and responsibility using an 8-dimension framework covering bias, secur... | 5/10 ★★ | 360 |
| [stop-testing-attacks-start](skills/stop-testing-attacks-start/SKILL.md) | Diagnose LLM safety defenses using the Four-Checkpoint Framework. Instead of asking "does this jailbreak work?", syst... | 5/10 ★★ | 222 |
| [the-shadow-self-intrinsic](skills/the-shadow-self-intrinsic/SKILL.md) | Detect and mitigate intrinsic value misalignment in LLM agent systems using the IMPRESS scenario-driven framework. Us... | 5/10 ★★ | 234 |
| [toward-universal-transferable-jailbreak](skills/toward-universal-transferable-jailbreak/SKILL.md) | Defend vision-language models (VLMs) against universal and transferable adversarial image attacks using techniques fr... | 5/10 ★★ | 317 |
| [triplay-rl-tri-role-self-play-reinforcement](skills/triplay-rl-tri-role-self-play-reinforcement/SKILL.md) | Apply the TriPlay-RL tri-role adversarial self-play framework to systematically red-team, harden, and evaluate LLM-po... | 5/10 ★★ | 208 |
| [alienlm-alienization-api-boundary-privacy](skills/alienlm-alienization-api-boundary-privacy/SKILL.md) | Implement AlienLM-style API-boundary privacy layers that protect sensitive text sent to black-box LLM APIs using voca... | 4/10 ★★ | 202 |
| [from-sparse-decisions-dense](skills/from-sparse-decisions-dense/SKILL.md) | Build content moderation and safety classification systems using multi-attribute trajectory reasoning instead of bina... | 4/10 ★★ | 261 |
| [malicious-repurposing-open-science](skills/malicious-repurposing-open-science/SKILL.md) | Defensive dual-use risk assessment for open science artifacts. Evaluates research papers, datasets, methods, and tool... | 4/10 ★★ | 212 |
| [semantic-aware-advanced-persistent-threat](skills/semantic-aware-advanced-persistent-threat/SKILL.md) | Build anomaly detection pipelines for Advanced Persistent Threat (APT) detection by encoding system logs into semanti... | 4/10 ★★ | 194 |

---

## Domain-Specific

**60 skills** | Avg rating: 5.1/10

| Skill | Description | Rating | Lines |
|-------|-------------|--------|-------|
| [ai-agent-for-reverseengineering](skills/ai-agent-for-reverseengineering/SKILL.md) | Reverse-engineer legacy numerical/scientific Fortran or C code and translate it into modern Python frameworks (Devito... | 7/10 ★★★★ | 205 |
| [canonical-intermediate-representation-llm-based](skills/canonical-intermediate-representation-llm-based/SKILL.md) | Translate natural language optimization problems into executable solver code using a Canonical Intermediate Represent... | 7/10 ★★★★ | 237 |
| [llm-fsm-scaling-finite-state-reasoning](skills/llm-fsm-scaling-finite-state-reasoning/SKILL.md) | Generate correct RTL (Verilog/SystemVerilog) implementations of finite-state machines from natural-language specifica... | 7/10 ★★★★ | 265 |
| [mirror-multi-agent-framework-iterative](skills/mirror-multi-agent-framework-iterative/SKILL.md) | Translate natural language optimization problems into mathematical models and solver code using MIRROR's multi-agent ... | 7/10 ★★★★ | 166 |
| [solagent-specialized-multi-agent-framework](skills/solagent-specialized-multi-agent-framework/SKILL.md) | Generate secure, functionally correct Solidity smart contracts using a dual-loop refinement process: an inner loop th... | 7/10 ★★★★ | 193 |
| [automating-computational-reproducibility-social](skills/automating-computational-reproducibility-social/SKILL.md) | Diagnose and repair failing computational research code to restore reproducibility. Uses an agent-based iterative wor... | 6/10 ★★★ | 189 |
| [benchmarking-abap-code-generation](skills/benchmarking-abap-code-generation/SKILL.md) | Generate syntactically correct and functional ABAP code using iterative compiler feedback loops. Applies the empirica... | 6/10 ★★★ | 183 |
| [can-implement-agent-based-odd-based](skills/can-implement-agent-based-odd-based/SKILL.md) | Translate ODD protocol specifications into validated, executable agent-based model (ABM) code in Python. Use when the... | 6/10 ★★★ | 208 |
| [echo-open-research-platform](skills/echo-open-research-platform/SKILL.md) | Build and configure ECHO-style research platforms for running reproducible user studies comparing chat-based AI and w... | 6/10 ★★★ | 207 |
| [gamedevbench-evaluating-agentic-capabilities](skills/gamedevbench-evaluating-agentic-capabilities/SKILL.md) | Agentic game development with visual feedback loops for Godot Engine projects. Applies the GameDevBench methodology: ... | 6/10 ★★★ | 187 |
| [localv-exploiting-information-locality](skills/localv-exploiting-information-locality/SKILL.md) | Multi-agent framework for generating large-scale Verilog/RTL code from long hardware specifications by decomposing lo... | 6/10 ★★★ | 138 |
| [opinf-llm-parametric-pde-solving](skills/opinf-llm-parametric-pde-solving/SKILL.md) | Solve parametric PDEs using operator inference with reduced-order models. Builds POD-based reduced dynamics from a sm... | 6/10 ★★★ | 200 |
| [rethinking-scientific-modeling-physically](skills/rethinking-scientific-modeling-physically/SKILL.md) | Generate physics-consistent, simulation-executable structural engineering code using constraint-oriented alignment an... | 6/10 ★★★ | 209 |
| [world-workflows-benchmark-bringing](skills/world-workflows-benchmark-bringing/SKILL.md) | Build world models for enterprise systems with hidden workflows and cascading database effects. Applies the probe-obs... | 6/10 ★★★ | 185 |
| [adoption-use-at-academic](skills/adoption-use-at-academic/SKILL.md) | Build institutional LLM platforms that integrate with existing data systems (EHR, CRM, ERP) using the ChatEHR pattern... | 5/10 ★★ | 389 |
| [ai-agent-systems-supply](skills/ai-agent-systems-supply/SKILL.md) | Build LLM-based multi-agent systems for supply chain inventory management using structured decision prompts and memor... | 5/10 ★★ | 277 |
| [alertguardian-intelligent-alert-life-cycle](skills/alertguardian-intelligent-alert-life-cycle/SKILL.md) | Build intelligent alert lifecycle management systems for cloud infrastructure using graph-based denoising, RAG-powere... | 5/10 ★★ | 201 |
| [alignagent-adaptive-learner-intelligence](skills/alignagent-adaptive-learner-intelligence/SKILL.md) | Build multi-agent adaptive learning systems that diagnose knowledge gaps and recommend targeted resources. Implements... | 5/10 ★★ | 319 |
| [analyticsgpt-workflow-scientometric-question](skills/analyticsgpt-workflow-scientometric-question/SKILL.md) | Build sequential LLM pipelines for scientometric question answering over academic databases. Decomposes meta-scientif... | 5/10 ★★ | 306 |
| [assessing-business-process-modeling](skills/assessing-business-process-modeling/SKILL.md) | Evaluate and generate BPMN process models from natural language using the BEF4LLM framework. Assess BPMN XML quality ... | 5/10 ★★ | 229 |
| [bridging-modality-gap-roadside](skills/bridging-modality-gap-roadside/SKILL.md) | Build training-free pipelines that convert sparse 3D LiDAR point clouds into depth-encoded 2D images for classificati... | 5/10 ★★ | 211 |
| [c2rope-causal-continuous-rotary-positional](skills/c2rope-causal-continuous-rotary-positional/SKILL.md) | Implement C²RoPE (Causal Continuous Rotary Positional Encoding) for multimodal transformers that process 2D/3D visual... | 5/10 ★★ | 235 |
| [chipbench-next-step-benchmark-evaluating](skills/chipbench-next-step-benchmark-evaluating/SKILL.md) | Evaluate and improve LLM-generated hardware designs using ChipBench methodology: structured Verilog generation with h... | 5/10 ★★ | 225 |
| [cognitive-platform-engineering-autonomous](skills/cognitive-platform-engineering-autonomous/SKILL.md) | Build autonomous cloud operations using a four-plane cognitive architecture (Sensing, Reasoning, Orchestration, Exper... | 5/10 ★★ | 247 |
| [dynamic-framework-collaborative-learning](skills/dynamic-framework-collaborative-learning/SKILL.md) | Build AI-moderated collaborative learning platforms with LLM-driven discussion facilitation, adaptive feedback, and p... | 5/10 ★★ | 214 |
| [evaluating-kubernetes-performance-genai](skills/evaluating-kubernetes-performance-genai/SKILL.md) | Design and optimize Kubernetes-native GenAI inference platforms using Kueue job queuing, Dynamic Accelerator Slicer (... | 5/10 ★★ | 217 |
| [eventcast-hybrid-demand-forecasting](skills/eventcast-hybrid-demand-forecasting/SKILL.md) | Build hybrid demand forecasting systems that fuse LLM-extracted event knowledge with time-series models using a dual-... | 5/10 ★★ | 228 |
| [evolve-evolutionary-search-llm-based](skills/evolve-evolutionary-search-llm-based/SKILL.md) | Evolutionary search framework for LLM-driven Verilog/RTL generation and PPA optimization. Uses MCTS for functional co... | 5/10 ★★ | 174 |
| [fin-rate-real-world-financial-analytics](skills/fin-rate-real-world-financial-analytics/SKILL.md) | Analyze SEC filings and financial disclosures using the Fin-RATE three-pathway methodology: detail-oriented reasoning... | 5/10 ★★ | 182 |
| [from-code-centric-concept-centric-teaching](skills/from-code-centric-concept-centric-teaching/SKILL.md) | Generate LLM-assisted coding labs that teach concepts through 'Vibe Coding' — producing working code paired with mand... | 5/10 ★★ | 269 |
| [from-gameplay-traces-game](skills/from-gameplay-traces-game/SKILL.md) | Reverse-engineer game mechanics from gameplay traces using a two-stage causal induction pipeline: first infer a Struc... | 5/10 ★★ | 211 |
| [from-pragmas-partners-symbiotic](skills/from-pragmas-partners-symbiotic/SKILL.md) | Agentic High-Level Synthesis (HLS) optimization: autonomously analyze, insert, and tune C/C++ HLS pragmas (pipeline, ... | 5/10 ★★ | 186 |
| [gamms-graph-based-adversarial](skills/gamms-graph-based-adversarial/SKILL.md) | Build and run graph-based multi-agent adversarial simulations using the GAMMS framework. Covers agent creation, graph... | 5/10 ★★ | 305 |
| [guideai-real-time-personalized-learning](skills/guideai-real-time-personalized-learning/SKILL.md) | Adaptive learning content generator that dynamically adjusts complexity, tone, pacing, and modality based on learner ... | 5/10 ★★ | 276 |
| [ic-eo-interpretable-code-based-assistant](skills/ic-eo-interpretable-code-based-assistant/SKILL.md) | Build conversational Earth Observation agents that turn natural-language queries into executable, auditable Python wo... | 5/10 ★★ | 215 |
| [llm-not-all-you](skills/llm-not-all-you/SKILL.md) | Systematic model selection advisor for classification tasks — chooses between classical ML, zero-shot LLMs/VLMs, and ... | 5/10 ★★ | 187 |
| [medbeads-agent-native-immutable-data](skills/medbeads-agent-native-immutable-data/SKILL.md) | Build immutable, agent-native medical data pipelines using Merkle DAG structures (MedBeads pattern). Converts mutable... | 5/10 ★★ | 182 |
| [medverse-reliable-medical-reasoning](skills/medverse-reliable-medical-reasoning/SKILL.md) | Decompose complex medical reasoning into DAG-structured parallel execution paths using Petri net theory. Improves acc... | 5/10 ★★ | 216 |
| [open-tutorai-open-source-platform](skills/open-tutorai-open-source-platform/SKILL.md) | Build personalized AI tutoring systems with structured onboarding, four-layer prompt architecture, adaptive lesson ge... | 5/10 ★★ | 266 |
| [openguandan-large-scale-imperfect-information](skills/openguandan-large-scale-imperfect-information/SKILL.md) | Build AI agents for the OpenGuanDan imperfect-information card game benchmark. Covers WebSocket client implementation... | 5/10 ★★ | 366 |
| [opportunities-aiml-rubin-lsst](skills/opportunities-aiml-rubin-lsst/SKILL.md) | Build trustworthy ML pipelines for large-scale scientific data analysis with calibrated uncertainties, simulation-bas... | 5/10 ★★ | 249 |
| [pcbschemagen-constraint-guided-schematic-design](skills/pcbschemagen-constraint-guided-schematic-design/SKILL.md) | Generate PCB schematics from natural language using constraint-guided LLM code generation with knowledge-graph verifi... | 5/10 ★★ | 209 |
| [proopf-benchmarking-improving-professional-grade](skills/proopf-benchmarking-improving-professional-grade/SKILL.md) | Translate natural-language power system operational requirements into executable Optimal Power Flow (OPF) optimizatio... | 5/10 ★★ | 218 |
| [synthagent-multi-agent-framework-realistic](skills/synthagent-multi-agent-framework-realistic/SKILL.md) | Build multi-agent pipelines that generate realistic synthetic patient profiles by integrating epidemiological data, m... | 5/10 ★★ | 298 |
| [unit-based-agent-semi-cascaded-full-duplex](skills/unit-based-agent-semi-cascaded-full-duplex/SKILL.md) | Build full-duplex voice dialogue systems using unit-based agent decomposition and semi-cascaded pipelines. Trigger ph... | 5/10 ★★ | 247 |
| [veri-sure-contract-aware-multi-agent-framework](skills/veri-sure-contract-aware-multi-agent-framework/SKILL.md) | Generate functionally correct RTL/Verilog code using a contract-aware multi-agent workflow with formal verification. ... | 5/10 ★★ | 186 |
| [yoloe-26-integrating-yolo26-yoloe](skills/yoloe-26-integrating-yolo26-yoloe/SKILL.md) | Build and deploy real-time open-vocabulary instance segmentation pipelines using YOLOE-26, which combines YOLOv26's N... | 5/10 ★★ | 232 |
| [agentic-ai-healthcare-medicine](skills/agentic-ai-healthcare-medicine/SKILL.md) | Design, evaluate, and improve LLM-based agentic systems for healthcare using a seven-dimensional taxonomy with 29 sub... | 4/10 ★★ | 274 |
| [arkeval-benchmarking-evaluating-automated](skills/arkeval-benchmarking-evaluating-automated/SKILL.md) | Automated ArkTS code repair using retrieval-augmented generation, LLM-based test oracle synthesis, and structured ben... | 4/10 ★★ | 196 |
| [mathliblemma-folklore-lemma-generation](skills/mathliblemma-folklore-lemma-generation/SKILL.md) | Multi-agent system for discovering and formalizing missing 'folklore' lemmas in Lean 4 / Mathlib. Identifies gaps in ... | 4/10 ★★ | 181 |
| [mdl-unified-multi-distribution-learner](skills/mdl-unified-multi-distribution-learner/SKILL.md) | Design and implement MDL (Multi-Distribution Learner) architectures for industrial recommendation systems that jointl... | 4/10 ★★ | 177 |
| [omnireview-large-scale-benchmark-llm-enhanced](skills/omnireview-large-scale-benchmark-llm-enhanced/SKILL.md) | Build reviewer/expert recommendation systems using LLM-generated semantic profiles and Multi-gate Mixture-of-Experts ... | 4/10 ★★ | 204 |
| [quasar-universal-autonomous-system](skills/quasar-universal-autonomous-system/SKILL.md) | Build autonomous multi-scale scientific simulation pipelines using the QUASAR architecture: a Strategist-Operator-Eva... | 4/10 ★★ | 165 |
| [realhd-high-quality-dataset-robust](skills/realhd-high-quality-dataset-robust/SKILL.md) | Detect AI-generated images using NLM noise entropy analysis and build robust forensic detection pipelines. Use when: ... | 4/10 ★★ | 230 |
| [report-nsf-workshop-ai](skills/report-nsf-workshop-ai/SKILL.md) | Apply AI techniques from the NSF AI-for-EDA workshop to hardware design tasks: RTL code generation from natural langu... | 4/10 ★★ | 276 |
| [scratcheval-multimodal-evaluation-framework](skills/scratcheval-multimodal-evaluation-framework/SKILL.md) | Evaluate, debug, and repair block-based Scratch programs using a three-layer executable protocol (VM execution, block... | 4/10 ★★ | 156 |
| [teaching-evaluating-reason-about](skills/teaching-evaluating-reason-about/SKILL.md) | Apply knowledge-augmented reasoning distillation for polymer design tasks. Builds structured Chain-of-Thought pipelin... | 4/10 ★★ | 202 |
| [visual-cognitive-demands-model-powered](skills/visual-cognitive-demands-model-powered/SKILL.md) | Evaluate visual and cognitive demands of in-vehicle LLM interfaces using the Monk et al. (2026) dual-metric framework... | 4/10 ★★ | 290 |
| [eft-cot-multi-agent-chain-of-thought-framework](skills/eft-cot-multi-agent-chain-of-thought-framework/SKILL.md) | Build multi-agent emotion-focused therapy (EFT) reasoning pipelines for empathetic mental health Q&A systems. Uses a ... | 3/10 ★★ | 300 |
| [magellan-autonomous-discovery-compiler](skills/magellan-autonomous-discovery-compiler/SKILL.md) | Evolve compiler optimization heuristics by coupling LLM code generation with evolutionary search and autotuning. Synt... | 3/10 ★★ | 158 |

---

## Agentic Systems

**54 skills** | Avg rating: 5.7/10

| Skill | Description | Rating | Lines |
|-------|-------------|--------|-------|
| [when-agents-fail-comprehensive](skills/when-agents-fail-comprehensive/SKILL.md) | Diagnose and fix bugs in LLM agent systems using a research-backed taxonomy of 11 bug types, 9 root causes, and 12 ob... | 8/10 ★★★★ | 240 |
| [a-mapreduce-executing-wide-search](skills/a-mapreduce-executing-wide-search/SKILL.md) | Execute large-scale breadth-oriented search and retrieval tasks using the A-MapReduce pattern: decompose a wide query... | 7/10 ★★★★ | 196 |
| [agentstepper-interactive-debugging-software](skills/agentstepper-interactive-debugging-software/SKILL.md) | Interactive debugging of LLM-powered software development agents using structured trajectory analysis, stepwise execu... | 7/10 ★★★★ | 197 |
| [agenttrace-structured-logging-framework](skills/agenttrace-structured-logging-framework/SKILL.md) | Implement structured, multi-surface observability logging for LLM agent systems using the AgentTrace pattern: operati... | 7/10 ★★★★ | 152 |
| [ci4a-semantic-component-interfaces](skills/ci4a-semantic-component-interfaces/SKILL.md) | Build semantic component interfaces that expose UI components as structured tool primitives for AI agent automation. ... | 7/10 ★★★★ | 272 |
| [from-task-solving-robust](skills/from-task-solving-robust/SKILL.md) | Build LLM agent workflows that stay robust under partial observability, noisy signals, shifting environments, and int... | 7/10 ★★★★ | 199 |
| [fs-researcher-test-time-scaling-long-horizon](skills/fs-researcher-test-time-scaling-long-horizon/SKILL.md) | File-system-based dual-agent deep research framework that scales beyond context windows. Separates evidence gathering... | 7/10 ★★★★ | 221 |
| [llm-in-sandbox-elicits-general-agentic](skills/llm-in-sandbox-elicits-general-agentic/SKILL.md) | Solve non-code tasks (math, science, long-context, formatting) by treating the terminal as a sandbox for exploration:... | 7/10 ★★★★ | 198 |
| [pabu-progress-aware-belief-update](skills/pabu-progress-aware-belief-update/SKILL.md) | Apply Progress-Aware Belief Update (PABU) to build efficient LLM agents that track task progress and selectively reta... | 7/10 ★★★★ | 210 |
| [pearl-plan-exploration-adaptive](skills/pearl-plan-exploration-adaptive/SKILL.md) | Apply PEARL's two-phase tool orchestration: offline tool exploration to learn valid usage patterns and failure modes,... | 7/10 ★★★★ | 172 |
| [rethinking-value-agent-generated-tests](skills/rethinking-value-agent-generated-tests/SKILL.md) | Optimize agent test-writing strategy for issue resolution by reallocating interaction budget from excessive test gene... | 7/10 ★★★★ | 163 |
| [roma-recursive-open-meta-agent](skills/roma-recursive-open-meta-agent/SKILL.md) | Decompose long-horizon, multi-step tasks using ROMA's recursive meta-agent pattern: Atomizer decides if a task needs ... | 7/10 ★★★★ | 185 |
| [table-as-search-formulate-long-horizon-agentic](skills/table-as-search-formulate-long-horizon-agentic/SKILL.md) | Structured table-completion framework for long-horizon information seeking. Converts complex research queries into da... | 7/10 ★★★★ | 203 |
| [adareasoner-dynamic-tool-orchestration](skills/adareasoner-dynamic-tool-orchestration/SKILL.md) | Adaptive multi-step tool orchestration for complex reasoning tasks. Dynamically selects, sequences, and composes tool... | 6/10 ★★★ | 168 |
| [avenir-web-human-experience-imitating-multimodal-w](skills/avenir-web-human-experience-imitating-multimodal-w/SKILL.md) | Build robust web automation agents using Mixture of Grounding Experts, experience-imitation planning, and task-tracki... | 6/10 ★★★ | 375 |
| [beyond-accuracy-cognitive-load](skills/beyond-accuracy-cognitive-load/SKILL.md) | Analyze and reduce cognitive load in tool-use agent workflows using the Cognitive Load Framework from AAAI 2026. Diag... | 6/10 ★★★ | 218 |
| [cua-skill-develop-skills-computer](skills/cua-skill-develop-skills-computer/SKILL.md) | Build reusable, parameterized skill libraries for computer-using agents (CUAs). Decomposes GUI automation into Skill ... | 6/10 ★★★ | 297 |
| [dllm-agent-see-farther](skills/dllm-agent-see-farther/SKILL.md) | Design and implement multi-agent workflows using the DeepDiver hierarchical orchestration pattern with diffusion-insp... | 6/10 ★★★ | 163 |
| [from-prompt-response-goal-directed-systems](skills/from-prompt-response-goal-directed-systems/SKILL.md) | Design production-grade agentic AI architectures with separated cognition/execution layers, typed tool interfaces, mu... | 6/10 ★★★ | 177 |
| [learning-irrecoverable-error-localized-policy](skills/learning-irrecoverable-error-localized-policy/SKILL.md) | Debug multi-step tool-using agent pipelines by localizing the first irrecoverable error via binary-search rollback, t... | 6/10 ★★★ | 175 |
| [mitigating-conversational-inertia-multi-turn](skills/mitigating-conversational-inertia-multi-turn/SKILL.md) | Detect and break conversational inertia in multi-turn agent interactions — where an LLM repeats its own prior actions... | 6/10 ★★★ | 236 |
| [pathwise-planning-world-automated](skills/pathwise-planning-world-automated/SKILL.md) | Multi-agent heuristic design framework that uses an entailment graph, policy/world-model/critic agents, and routed re... | 6/10 ★★★ | 165 |
| [planner-auditor-twin-agentic-discharge](skills/planner-auditor-twin-agentic-discharge/SKILL.md) | Implement a Planner-Auditor twin architecture that decouples LLM generation from deterministic validation with self-i... | 6/10 ★★★ | 189 |
| [smartoracle-agentic-approach](skills/smartoracle-agentic-approach/SKILL.md) | Agentic differential oracle for triaging cross-implementation discrepancies. Decomposes bug triage into specialized s... | 6/10 ★★★ | 177 |
| [strong-reasoning-isnt-enough](skills/strong-reasoning-isnt-enough/SKILL.md) | Build interactive diagnostic agents that systematically elicit evidence before concluding, using the REFINE (Reasonin... | 6/10 ★★★ | 251 |
| [toolself-unifying-task-execution](skills/toolself-unifying-task-execution/SKILL.md) | Implement self-reconfiguring agent workflows where configuration (sub-goals, strategy, toolbox, context) is a mutable... | 6/10 ★★★ | 217 |
| [towards-declarative-agentic-layer](skills/towards-declarative-agentic-layer/SKILL.md) | Build grounded, declarative agentic architectures using the DALIA pattern: capability descriptors, discovery protocol... | 6/10 ★★★ | 218 |
| [when-agents-fail-act](skills/when-agents-fail-act/SKILL.md) | Diagnose and fix tool invocation failures in multi-agent LLM systems using a 12-category error taxonomy covering tool... | 6/10 ★★★ | 194 |
| [yunque-deepresearch-technical-report](skills/yunque-deepresearch-technical-report/SKILL.md) | Hierarchical multi-agent deep research framework with dynamic context management and supervisor-based error recovery.... | 6/10 ★★★ | 191 |
| [acegrpo-adaptive-curriculum-group](skills/acegrpo-adaptive-curriculum-group/SKILL.md) | Adaptive curriculum-driven iterative optimization for autonomous ML engineering tasks. Uses Evolving Data Buffers and... | 5/10 ★★ | 220 |
| [agentic-very-long-video](skills/agentic-very-long-video/SKILL.md) | Build agentic systems for understanding very long video streams (hours to weeks) using entity scene graphs, multi-too... | 5/10 ★★ | 240 |
| [ai-my-values-user](skills/ai-my-values-user/SKILL.md) | Build value-aligned conversational agents using the VAPT (Value-Alignment Perception Toolkit) framework from CHI '26.... | 5/10 ★★ | 236 |
| [alrm-agentic-robotic-manipulation](skills/alrm-agentic-robotic-manipulation/SKILL.md) | Build agentic LLM-driven robotic manipulation pipelines using the ALRM framework pattern: a ReAct-style reasoning loo... | 5/10 ★★ | 131 |
| [bayesflow-probability-inference-framework](skills/bayesflow-probability-inference-framework/SKILL.md) | Generate high-quality multi-step LLM workflows using Bayesian inference with parallel look-ahead rollouts and importa... | 5/10 ★★ | 204 |
| [curiosity-driven-knowledge-retrieval](skills/curiosity-driven-knowledge-retrieval/SKILL.md) | Implements a curiosity-driven knowledge retrieval framework for autonomous agents. Formalizes agent uncertainty as a ... | 5/10 ★★ | 219 |
| [deep-search-hierarchical-meta-cognitive](skills/deep-search-hierarchical-meta-cognitive/SKILL.md) | Implement hierarchical meta-cognitive monitoring for deep search agents. Embeds a two-tier self-monitoring system (fa... | 5/10 ★★ | 188 |
| [deepplanning-benchmarking-long-horizon-agentic](skills/deepplanning-benchmarking-long-horizon-agentic/SKILL.md) | Solve long-horizon planning tasks with verifiable constraints using the DeepPlanning methodology: proactive informati... | 5/10 ★★ | 155 |
| [effgen-enabling-small-language](skills/effgen-enabling-small-language/SKILL.md) | Deploy and optimize small language models (SLMs) as autonomous agents using the effGen framework. Implements prompt c... | 5/10 ★★ | 193 |
| [evolving-tool-user-creator](skills/evolving-tool-user-creator/SKILL.md) | Transform Claude from a static tool user into a dynamic tool creator using the UCT (User-to-Creator Transformation) f... | 5/10 ★★ | 181 |
| [fat-cat-document-driven-metacognitive-multi-agent](skills/fat-cat-document-driven-metacognitive-multi-agent/SKILL.md) | Implement the Fat-Cat document-driven metacognitive agent architecture for complex multi-step reasoning tasks. Uses M... | 5/10 ★★ | 225 |
| [from-passive-metric-active](skills/from-passive-metric-active/SKILL.md) | Build systems that use LLM uncertainty as an active control signal -- routing computation, triggering tool calls, ena... | 5/10 ★★ | 268 |
| [gisa-benchmark-general-information-seeking](skills/gisa-benchmark-general-information-seeking/SKILL.md) | Build structured information-seeking agents that decompose complex queries into multi-turn search-and-browse workflow... | 5/10 ★★ | 162 |
| [haif-human-ai-integration-framework](skills/haif-human-ai-integration-framework/SKILL.md) | Apply the HAIF protocol to organize hybrid human-AI team workflows with tiered autonomy, delegation governance, and v... | 5/10 ★★ | 206 |
| [just-in-time-reinforcement-learning-continual](skills/just-in-time-reinforcement-learning-continual/SKILL.md) | Implement JitRL-style continual learning for LLM agents: training-free policy optimization via experience memory, adv... | 5/10 ★★ | 203 |
| [learning-compose-cross-domain-agentic](skills/learning-compose-cross-domain-agentic/SKILL.md) | Generate cross-domain agentic workflows using decompose-recompose-decide composition over reusable capability bases. ... | 5/10 ★★ | 159 |
| [longcat-flash-thinking-2601-technical-report](skills/longcat-flash-thinking-2601-technical-report/SKILL.md) | Build robust multi-tool agentic pipelines with noise-aware execution, parallel reasoning, and environment scaling pat... | 5/10 ★★ | 311 |
| [skillrl-evolving-agents-recursive](skills/skillrl-evolving-agents-recursive/SKILL.md) | Build self-improving agent systems that distill raw execution traces into a hierarchical skill library (SkillBank) an... | 5/10 ★★ | 184 |
| [supchain-bench-benchmarking-real-world-supply](skills/supchain-bench-benchmarking-real-world-supply/SKILL.md) | Build reliable long-horizon supply chain agents using the SupChain-ReAct pattern: multi-path ReAct trajectories with ... | 5/10 ★★ | 198 |
| [vln-pilot-vision-language-as-autonomous](skills/vln-pilot-vision-language-as-autonomous/SKILL.md) | Build VLLM-driven autonomous navigation agents that interpret natural language instructions and ground them in visual... | 5/10 ★★ | 235 |
| [darwin-dynamic-agentically-rewriting](skills/darwin-dynamic-agentically-rewriting/SKILL.md) | Evolutionary multi-agent code optimization using genetic algorithms. Agents mutate each other's training/configuratio... | 4/10 ★★ | 167 |
| [embodied-task-planning-graph-informed](skills/embodied-task-planning-graph-informed/SKILL.md) | Structure long-horizon task planning using graph-based memory and bounded lookahead. Use when asked to: 'plan a multi... | 4/10 ★★ | 179 |
| [farm-field-aware-resolution-intelligent](skills/farm-field-aware-resolution-intelligent/SKILL.md) | Build intelligent trigger-action automation systems using FARM's two-stage architecture: contrastive retrieval + mult... | 4/10 ★★ | 187 |
| [perfguard-performance-aware-agent-visual](skills/perfguard-performance-aware-agent-visual/SKILL.md) | Performance-aware multi-tool orchestration for visual content generation pipelines. Implements PerfGuard's three mech... | 4/10 ★★ | 227 |
| [t2vtree-user-centered-visual-analytics](skills/t2vtree-user-centered-visual-analytics/SKILL.md) | Build tree-structured, agent-assisted thought-to-video authoring systems where each generation step is a node binding... | 4/10 ★★ | 268 |

---

## Code & Software Engineering

**46 skills** | Avg rating: 6.6/10

| Skill | Description | Rating | Lines |
|-------|-------------|--------|-------|
| [davinci-agency-unlocking-long-horizon-agency](skills/davinci-agency-unlocking-long-horizon-agency/SKILL.md) | Decompose complex, long-horizon coding tasks into PR-like chains of verifiable subtasks with cross-stage dependency t... | 8/10 ★★★★ | 181 |
| [docksmith-scaling-reliable-coding](skills/docksmith-scaling-reliable-coding/SKILL.md) | Build reliable Docker environments for arbitrary code repositories using an agentic, multi-phase approach with depend... | 8/10 ★★★★ | 201 |
| [environment-in-the-loop-rethinking-code-migration-](skills/environment-in-the-loop-rethinking-code-migration/SKILL.md) | Perform code migrations (dependency upgrades, API changes, framework transitions) with integrated environment verific... | 8/10 ★★★★ | 162 |
| [ide-bench-evaluating-as-ide](skills/ide-bench-evaluating-as-ide/SKILL.md) | Apply IDE-Bench's structured agent workflow for tackling real-world software engineering tasks: systematic exploratio... | 8/10 ★★★★ | 149 |
| [spell-synthesis-programmatic-edits](skills/spell-synthesis-programmatic-edits/SKILL.md) | Automate library migrations by synthesizing reusable code transformation scripts. Uses LLM-generated migration exampl... | 8/10 ★★★★ | 209 |
| [test-vs-mutant-adversarial](skills/test-vs-mutant-adversarial/SKILL.md) | Adversarial test generation using two competing LLM agents: a Test Agent that writes unit tests and a Mutant Agent th... | 8/10 ★★★★ | 221 |
| [aacr-bench-evaluating-automatic-code](skills/aacr-bench-evaluating-automatic-code/SKILL.md) | Perform repository-level automated code review on pull requests using hierarchical context retrieval and structured d... | 7/10 ★★★★ | 187 |
| [beyond-blame-rethinking-szz](skills/beyond-blame-rethinking-szz/SKILL.md) | Identify bug-inducing commits using temporal knowledge graph search beyond git blame. Use when: 'find what commit int... | 7/10 ★★★★ | 154 |
| [can-we-classify-flaky](skills/can-we-classify-flaky/SKILL.md) | Analyze test suites for flaky tests using LLM-based classification with context-augmented reasoning. Applies findings... | 7/10 ★★★★ | 182 |
| [consistency-meets-verification-enhancing](skills/consistency-meets-verification-enhancing/SKILL.md) | Generate high-reliability test suites without ground-truth implementations using the ConVerTest pipeline: Self-Consis... | 7/10 ★★★★ | 178 |
| [davinci-dev-agent-native-mid-training-software](skills/davinci-dev-agent-native-mid-training-software/SKILL.md) | Apply daVinci-Dev's agent-native workflow to software engineering tasks: navigate repos, localize bugs, plan edits, a... | 7/10 ★★★★ | 171 |
| [detecting-correcting-hallucinations-llm-generated](skills/detecting-correcting-hallucinations-llm-generated/SKILL.md) | Detect and auto-correct hallucinated API calls in LLM-generated Python code using deterministic AST analysis and libr... | 7/10 ★★★★ | 260 |
| [evaluating-achieving-controllable-code](skills/evaluating-achieving-controllable-code/SKILL.md) | Instruction-guided code completion that follows user constraints on algorithm choice, data structures, control flow, ... | 7/10 ★★★★ | 197 |
| [evocodebench-human-performance-benchmark-self-evol](skills/evocodebench-human-performance-benchmark-self-evol/SKILL.md) | Self-evolving code generation with iterative reflection and revision. Applies a feedback-driven loop where code is su... | 7/10 ★★★★ | 174 |
| [fullstack-agent-enhancing-agentic-fullstack](skills/fullstack-agent-enhancing-agentic-fullstack/SKILL.md) | Build production-grade full-stack web applications using a three-agent pipeline (Planning, Backend, Frontend) with de... | 7/10 ★★★★ | 146 |
| [menvagent-scalable-polyglot-environment](skills/menvagent-scalable-polyglot-environment/SKILL.md) | Automated Docker environment construction for polyglot repositories using a Planning-Execution-Verification multi-age... | 7/10 ★★★★ | 188 |
| [more-code-less-reuse](skills/more-code-less-reuse/SKILL.md) | Analyze AI-generated code for redundancy and missed reuse opportunities using semantic clone detection, then refactor... | 7/10 ★★★★ | 187 |
| [on-impact-code-comments](skills/on-impact-code-comments/SKILL.md) | Use code comments as a bug-fixing amplifier: generate implementation-detail comments on buggy code before attempting ... | 7/10 ★★★★ | 219 |
| [precision-practice-knowledge-guided](skills/precision-practice-knowledge-guided/SKILL.md) | Generate industrial-grade code summaries using the ExpSum knowledge-guided approach: function metadata extraction, do... | 7/10 ★★★★ | 298 |
| [pull-requests-as-training](skills/pull-requests-as-training/SKILL.md) | Apply the Clean-PR agentless repo-level code editing protocol: decompose issues into file localization, fine-grained ... | 7/10 ★★★★ | 199 |
| [revisiting-role-natural-code](skills/revisiting-role-natural-code/SKILL.md) | Comment-augmented code translation (COMMENTRA) that uses targeted natural language comment injection to significantly... | 7/10 ★★★★ | 215 |
| [swe-agi-benchmarking-specification-driven-software](skills/swe-agi-benchmarking-specification-driven-software/SKILL.md) | Build production-scale software systems from formal specifications, RFCs, and standards documents using specification... | 7/10 ★★★★ | 189 |
| [swe-bench-mobile-agents-develop](skills/swe-bench-mobile-agents-develop/SKILL.md) | Apply defensive programming and agent-architecture patterns from SWE-Bench Mobile to tackle production iOS/mobile dev... | 7/10 ★★★★ | 247 |
| [swe-master-unleashing-potential-software](skills/swe-master-unleashing-potential-software/SKILL.md) | Applies the SWE-Master agentic software engineering methodology to solve complex, multi-file bugs and feature request... | 7/10 ★★★★ | 195 |
| [swe-refactor-repository-level-benchmark-real-world](skills/swe-refactor-repository-level-benchmark-real-world/SKILL.md) | Perform repository-level code refactoring with semantics-preserving guarantees using the SWE-Refactor methodology. Su... | 7/10 ★★★★ | 151 |
| [synthesizing-file-level-data-unit](skills/synthesizing-file-level-data-unit/SKILL.md) | Generate high-quality unit tests with self-debugging repair loops and chain-of-thought reasoning. Produces tests with... | 7/10 ★★★★ | 222 |
| [tam-eval-evaluating-llms-for](skills/tam-eval-evaluating-llms-for/SKILL.md) | Systematic test suite maintenance using the TAM-Eval framework's three-scenario approach: creating new tests, repairi... | 7/10 ★★★★ | 225 |
| [timemachine-bench-benchmark-evaluating-capabilitie](skills/timemachine-bench-benchmark-evaluating-capabilitie/SKILL.md) | Systematic dependency migration for Python projects. Diagnose and fix test failures caused by dependency updates usin... | 7/10 ★★★★ | 143 |
| [tracecoder-trace-driven-multi-agent-framework](skills/tracecoder-trace-driven-multi-agent-framework/SKILL.md) | Trace-driven debugging framework for LLM-generated code. Uses diagnostic probe instrumentation, causal trace analysis... | 7/10 ★★★★ | 191 |
| [variability-aware-detection-repair-compilation-err](skills/variability-aware-detection-repair-compilation-err/SKILL.md) | Detect and repair compilation errors hidden behind #ifdef/#ifndef/#if defined() preprocessor directives in configurab... | 7/10 ★★★★ | 180 |
| [where-ai-coding-agents](skills/where-ai-coding-agents/SKILL.md) | Pre-flight checker that prevents AI coding agent PRs from failing, based on empirical analysis of 33k agent-authored ... | 7/10 ★★★★ | 193 |
| [batcoder-self-supervised-bidirectional-code-docume](skills/batcoder-self-supervised-bidirectional-code-docume/SKILL.md) | Apply BatCoder's back-translation technique to improve code and documentation quality bidirectionally. Generate docum... | 6/10 ★★★ | 207 |
| [codecircuit-inferring-llm-generated-code](skills/codecircuit-inferring-llm-generated-code/SKILL.md) | Assess LLM-generated code correctness using attribution graph analysis inspired by mechanistic interpretability. Appl... | 6/10 ★★★ | 195 |
| [devops-gym-benchmarking-ai-agents](skills/devops-gym-benchmarking-ai-agents/SKILL.md) | Apply the DevOps-Gym methodology to systematically tackle full-cycle DevOps tasks: build/configuration repair, runtim... | 6/10 ★★★ | 170 |
| [doc2spec-synthesizing-formal-programming](skills/doc2spec-synthesizing-formal-programming/SKILL.md) | Synthesize formal programming specifications from natural-language API docs using grammar induction. Extracts rules f... | 6/10 ★★★ | 177 |
| [failure-aware-enhancements-code-generation](skills/failure-aware-enhancements-code-generation/SKILL.md) | Diagnose why generated code fails and apply the right fix strategy (self-critique, RAG, multi-model, or progressive p... | 6/10 ★★★ | 191 |
| [identifying-concurrency-bug-reports](skills/identifying-concurrency-bug-reports/SKILL.md) | Classify bug reports as concurrency-related using a four-level linguistic pattern taxonomy (word, phrase, sentence, r... | 6/10 ★★★ | 185 |
| [ral-bench-benchmarking-application-level-functiona](skills/ral-bench-benchmarking-application-level-functiona/SKILL.md) | Generate and evaluate complete multi-file application repositories with both functional correctness and non-functiona... | 6/10 ★★★ | 179 |
| [tricky2-benchmark-evaluating-human-error](skills/tricky2-benchmark-evaluating-human-error/SKILL.md) | Taxonomy-guided analysis of mixed human+LLM bugs in code. Classifies bug origins, localizes interacting defects, and ... | 6/10 ★★★ | 207 |
| [whitespaces-dont-lie-feature-driven](skills/whitespaces-dont-lie-feature-driven/SKILL.md) | Detect whether source code was written by a human or generated by an AI (ChatGPT, Copilot, etc.) using whitespace, in... | 6/10 ★★★ | 255 |
| [neural-theorem-proving-verification](skills/neural-theorem-proving-verification/SKILL.md) | Generate formal proofs for program verification conditions (VCs) in Isabelle, Lean 4, and Rocq. Translates C/WhyML co... | 5/10 ★★ | 202 |
| [reducing-costs-proof-synthesis](skills/reducing-costs-proof-synthesis/SKILL.md) | Generate formally verified Rust code with Verus specifications and proofs using the VeruSyn methodology. Applies self... | 5/10 ★★ | 247 |
| [supporting-software-engineering-tasks](skills/supporting-software-engineering-tasks/SKILL.md) | Generate test scenarios from requirements and retrieve/analyze software engineering documents using a supervisor-work... | 5/10 ★★ | 169 |
| [will-it-survive-deciphering](skills/will-it-survive-deciphering/SKILL.md) | Analyze the survival and maintenance fate of AI-generated code in repositories using survival analysis techniques fro... | 5/10 ★★ | 173 |
| [artificial-intelligence-open-source](skills/artificial-intelligence-open-source/SKILL.md) | Analyze open-source projects for sustainability risks and apply AI-driven interventions for bug triaging, community h... | 4/10 ★★ | 210 |
| [predicting-intermittent-job-failure](skills/predicting-intermittent-job-failure/SKILL.md) | Classify and diagnose intermittent CI/CD job failures from execution logs using the FlaXifyer few-shot approach and L... | 4/10 ★★ | 280 |

---

## Reasoning & Chain-of-Thought

**46 skills** | Avg rating: 5.7/10

| Skill | Description | Rating | Lines |
|-------|-------------|--------|-------|
| [bridging-arithmetic-gap-cognitive](skills/bridging-arithmetic-gap-cognitive/SKILL.md) | Iterative Dual-Phase Financial-PoT: decouple semantic reasoning from arithmetic computation to eliminate calculation ... | 7/10 ★★★★ | 222 |
| [debugging-code-world](skills/debugging-code-world/SKILL.md) | Debug code by mentally simulating execution as a Code World Model — predicting runtime state after each statement, ca... | 7/10 ★★★★ | 171 |
| [deep-researcher-sequential-plan](skills/deep-researcher-sequential-plan/SKILL.md) | Conduct deep, multi-step research on complex topics using Sequential Plan Refinement with Reflection and Candidates C... | 7/10 ★★★★ | 194 |
| [enhancing-mathematical-problem-solving](skills/enhancing-mathematical-problem-solving/SKILL.md) | Solve mathematical problems using IIPC (Iteratively Improved Program Construction) -- a dual-branch approach that com... | 7/10 ★★★★ | 183 |
| [funprm-function-as-step-process-reward](skills/funprm-function-as-step-process-reward/SKILL.md) | Generate high-quality code by decomposing solutions into modular functions (Chain-of-Function style), then self-evalu... | 7/10 ★★★★ | 251 |
| [monte-carlo-tree-search](skills/monte-carlo-tree-search/SKILL.md) | Apply Monte Carlo Tree Search (MCTS) to systematically explore and evaluate multiple fix candidates when debugging co... | 7/10 ★★★★ | 186 |
| [reasoning-while-asking-transforming](skills/reasoning-while-asking-transforming/SKILL.md) | Proactive Interactive Reasoning (PIR) — instead of guessing when requirements are ambiguous or premises are missing, ... | 7/10 ★★★★ | 190 |
| [swe-manager-selecting-synthesizing-golden](skills/swe-manager-selecting-synthesizing-golden/SKILL.md) | Select and synthesize a golden proposal from multiple candidate fix strategies before writing code. Mirrors how techn... | 7/10 ★★★★ | 254 |
| [wdscaling-parallel-tool-calling-deep](skills/wdscaling-parallel-tool-calling-deep/SKILL.md) | Scale deep research tasks by issuing parallel tool calls (width) alongside sequential reasoning (depth), following th... | 7/10 ★★★★ | 161 |
| [why-reasoning-fails-plan](skills/why-reasoning-fails-plan/SKILL.md) | Apply FLARE (Future-aware Lookahead with Reward Estimation) to long-horizon coding tasks. Replaces greedy step-by-ste... | 7/10 ★★★★ | 218 |
| [adaptive-confidence-gating-multi-agent](skills/adaptive-confidence-gating-multi-agent/SKILL.md) | Multi-agent code generation using structured debate with adaptive confidence gating. Three specialized agents (User/P... | 6/10 ★★★ | 184 |
| [chain-simulation-dual-mode-reasoning](skills/chain-simulation-dual-mode-reasoning/SKILL.md) | Dual-mode reasoning framework that dynamically routes problems to specialized strategies: computational flow for math... | 6/10 ★★★ | 175 |
| [contextual-drag-errors-context](skills/contextual-drag-errors-context/SKILL.md) | Mitigate contextual drag — the phenomenon where failed attempts in conversation context bias LLM reasoning toward str... | 6/10 ★★★ | 249 |
| [conversation-non-verifiable-learning-self-evolving](skills/conversation-non-verifiable-learning-self-evolving/SKILL.md) | Implements the CoNL (Conversation for Non-verifiable Learning) multi-agent self-play framework for iteratively improv... | 6/10 ★★★ | 241 |
| [convexbench-recognize-convex-functions](skills/convexbench-recognize-convex-functions/SKILL.md) | Determine the convexity of arbitrarily deep symbolic function compositions using AST decomposition and recursive DCP-... | 6/10 ★★★ | 161 |
| [corefine-confidence-guided-self-refinement-adaptiv](skills/corefine-confidence-guided-self-refinement-adaptiv/SKILL.md) | Confidence-guided self-refinement for adaptive reasoning. Implements the CoRefine pattern: assess confidence in each ... | 6/10 ★★★ | 176 |
| [dep-search-learning-dependency-aware-reasoning](skills/dep-search-learning-dependency-aware-reasoning/SKILL.md) | Dependency-aware multi-step reasoning with persistent memory for complex questions requiring information retrieval ac... | 6/10 ★★★ | 213 |
| [history-guided-iterative-visual-reasoning](skills/history-guided-iterative-visual-reasoning/SKILL.md) | Apply the H-GIVR (History-Guided Iterative Visual Reasoning) framework for self-correcting multimodal reasoning. Uses... | 6/10 ★★★ | 170 |
| [iesr-mcts-based-modular-reasoning](skills/iesr-mcts-based-modular-reasoning/SKILL.md) | Convert natural language questions into SQL queries using MCTS-based modular reasoning inspired by the IESR framework... | 6/10 ★★★ | 242 |
| [large-reasoning-failures](skills/large-reasoning-failures/SKILL.md) | Detect and mitigate known LLM reasoning failures during code generation, review, and problem-solving. Applies the tax... | 6/10 ★★★ | 218 |
| [prism-principled-framework-multi-agent](skills/prism-principled-framework-multi-agent/SKILL.md) | Apply the PRISM (Propose-Review-Integrate Synthesis) multi-agent reasoning framework to decompose hard problems into ... | 6/10 ★★★ | 138 |
| [proact-agentic-lookahead-interactive](skills/proact-agentic-lookahead-interactive/SKILL.md) | Apply ProAct-style lookahead reasoning to multi-step coding and planning tasks. Compresses search-tree exploration in... | 6/10 ★★★ | 241 |
| [rethinker-scientific-reasoning-rethinking](skills/rethinker-scientific-reasoning-rethinking/SKILL.md) | Solve hard scientific and technical reasoning problems using the ReThinker Solver-Critic-Selector loop with confidenc... | 6/10 ★★★ | 239 |
| [stalled-biased-confused-uncovering-reasoning](skills/stalled-biased-confused-uncovering-reasoning/SKILL.md) | Systematic root cause analysis for cloud/distributed system failures using a 16-category reasoning failure taxonomy a... | 6/10 ★★★ | 168 |
| [towards-autonomous-mathematics-research](skills/towards-autonomous-mathematics-research/SKILL.md) | Iterative generate-verify-revise agent for mathematical research problems. Implements the Aletheia loop: decompose a ... | 6/10 ★★★ | 239 |
| [aero-autonomous-evolutionary-reasoning](skills/aero-autonomous-evolutionary-reasoning/SKILL.md) | Apply the AERO dual-loop self-evolution framework to iteratively improve reasoning on complex tasks. Uses entropy-bas... | 5/10 ★★ | 160 |
| [can-post-training-transform-causal](skills/can-post-training-transform-causal/SKILL.md) | Perform rigorous causal inference tasks using structured reasoning pipelines inspired by CauGym. Estimate treatment e... | 5/10 ★★ | 179 |
| [causalt5k-diagnosing-informing-refusal](skills/causalt5k-diagnosing-informing-refusal/SKILL.md) | Diagnose and correct causal reasoning failures in LLM outputs using the CausalT5K framework. Detects rung collapse (a... | 5/10 ★★ | 161 |
| [chain-mindset-reasoning-adaptive](skills/chain-mindset-reasoning-adaptive/SKILL.md) | Solve complex problems by switching between four cognitive mindsets (Spatial, Convergent, Divergent, Algorithmic) at ... | 5/10 ★★ | 177 |
| [closing-reasoning-gaps-clinical](skills/closing-reasoning-gaps-clinical/SKILL.md) | Build systems that detect and fix reasoning gaps in LLM agents by comparing their chain-of-thought against reference ... | 5/10 ★★ | 196 |
| [ctrlcot-dual-granularity-chain-of-thought-compress](skills/ctrlcot-dual-granularity-chain-of-thought-compress/SKILL.md) | Compress chain-of-thought reasoning using CtrlCoT's dual-granularity framework: hierarchical semantic abstraction com... | 5/10 ★★ | 157 |
| [discovering-process-outcome-credit-multi-step](skills/discovering-process-outcome-credit-multi-step/SKILL.md) | Apply Step-wise Marginal Information Gain (MIG) credit assignment to multi-step reasoning tasks. Evaluates each reaso... | 5/10 ★★ | 190 |
| [empirical-mcts-continuous-agent-evolution](skills/empirical-mcts-continuous-agent-evolution/SKILL.md) | Applies Empirical-MCTS dual-loop reasoning: structured tree search with persistent memory that accumulates experience... | 5/10 ★★ | 195 |
| [latent-chain-of-thought-as-planning](skills/latent-chain-of-thought-as-planning/SKILL.md) | Decouple reasoning from verbalization using PLaT-inspired latent planning. Maintains a broad solution space through p... | 5/10 ★★ | 200 |
| [learning-reason-faithfully-step-level](skills/learning-reason-faithfully-step-level/SKILL.md) | Apply FaithRL's step-level faithfulness verification to multi-step reasoning tasks. Decomposes reasoning into individ... | 5/10 ★★ | 173 |
| [pope-learning-reason-hard](skills/pope-learning-reason-hard/SKILL.md) | Apply the POPE (Privileged On-Policy Exploration) technique to solve hard reasoning problems by decomposing them with... | 5/10 ★★ | 132 |
| [reasoning-beyond-literal-cross-style](skills/reasoning-beyond-literal-cross-style/SKILL.md) | Detect and interpret figurative language (sarcasm, humor, offense, metaphor) in multimodal image-text content using a... | 5/10 ★★ | 176 |
| [regular-variational-latent-reasoning](skills/regular-variational-latent-reasoning/SKILL.md) | Compress verbose chain-of-thought reasoning into compact latent state representations guided by rendered visual summa... | 5/10 ★★ | 236 |
| [self-hinting-enhance-reinforcement-learning](skills/self-hinting-enhance-reinforcement-learning/SKILL.md) | Apply the SAGE self-hinting technique to improve LLM problem-solving by generating graduated hints that boost solutio... | 5/10 ★★ | 174 |
| [state-transition-framework-reasoning](skills/state-transition-framework-reasoning/SKILL.md) | Applies a state-transition reasoning framework that models multi-step reasoning as an evolving state, compressing his... | 5/10 ★★ | 215 |
| [verge-formal-refinement-guidance](skills/verge-formal-refinement-guidance/SKILL.md) | Iterative verification-guided reasoning that decomposes answers into atomic claims, classifies and routes them to for... | 5/10 ★★ | 185 |
| [bridging-online-offline-rl](skills/bridging-online-offline-rl/SKILL.md) | Apply Cobalt-style contextual bandit learning to multi-turn code generation tasks. Decomposes iterative coding into p... | 4/10 ★★ | 160 |
| [egss-entropy-guided-stepwise-scaling](skills/egss-entropy-guided-stepwise-scaling/SKILL.md) | Apply Entropy-Guided Stepwise Scaling (EGSS) to complex software engineering tasks like bug fixing, code generation, ... | 4/10 ★★ | 164 |
| [s3-cot-self-sampled-succinct-reasoning](skills/s3-cot-self-sampled-succinct-reasoning/SKILL.md) | Apply dual-cognitive reasoning (System 1 fast / System 2 slow) to compress verbose chain-of-thought into succinct, ef... | 4/10 ★★ | 201 |
| [unicog-uncovering-cognitive-abilities](skills/unicog-uncovering-cognitive-abilities/SKILL.md) | Analyze and diagnose LLM reasoning through latent cognitive ability decomposition inspired by the UniCog framework. D... | 4/10 ★★ | 178 |
| [unveiling-cognitive-compass-theory-of-mind-guided](skills/unveiling-cognitive-compass-theory-of-mind-guided/SKILL.md) | Apply Theory-of-Mind (ToM) guided reasoning chains to multimodal emotion analysis tasks. Decomposes emotional reasoni... | 4/10 ★★ | 197 |

---

## Multi-Agent Systems

**45 skills** | Avg rating: 5.3/10

| Skill | Description | Rating | Lines |
|-------|-------------|--------|-------|
| [agyn-multi-agent-system-team-based](skills/agyn-multi-agent-system-team-based/SKILL.md) | Orchestrate multi-agent teams for autonomous software engineering using the Agyn methodology: coordinator, researcher... | 7/10 ★★★★ | 202 |
| [aorchestra-automating-sub-agent-creation](skills/aorchestra-automating-sub-agent-creation/SKILL.md) | Dynamically create specialized sub-agents for complex multi-step tasks using the AOrchestra pattern: decompose goals,... | 7/10 ★★★★ | 203 |
| [marti-mars2-scaling-multi-agent-self-search-reinfo](skills/marti-mars2-scaling-multi-agent-self-search-reinfo/SKILL.md) | Multi-agent tree-search code generation using heterogeneous agent collaboration with error-feedback refinement. Spawn... | 7/10 ★★★★ | 204 |
| [multi-agent-teams-hold-experts](skills/multi-agent-teams-hold-experts/SKILL.md) | Prevent expertise dilution in multi-agent LLM workflows by applying findings from 'Multi-Agent Teams Hold Experts Bac... | 7/10 ★★★★ | 151 |
| [wideseek-r1-exploring-width-scaling](skills/wideseek-r1-exploring-width-scaling/SKILL.md) | Decompose broad information-seeking tasks into parallel subtasks using a lead-agent-subagent pattern with isolated co... | 7/10 ★★★★ | 197 |
| [agent-primitives-reusable-latent](skills/agent-primitives-reusable-latent/SKILL.md) | Design and orchestrate multi-agent systems using reusable Agent Primitives (Review, Voting/Selection, Planning/Execut... | 6/10 ★★★ | 252 |
| [blind-gods-broken-screens](skills/blind-gods-broken-screens/SKILL.md) | Architect secure, intent-centric agent systems using the Aura pattern: Hub-and-Spoke agent topology, cryptographic id... | 6/10 ★★★ | 219 |
| [constrained-process-maps-multi-agent](skills/constrained-process-maps-multi-agent/SKILL.md) | Build multi-agent workflows structured as constrained DAG process maps with Monte Carlo uncertainty estimation. Each ... | 6/10 ★★★ | 244 |
| [dynamic-role-assignment-multi-agent](skills/dynamic-role-assignment-multi-agent/SKILL.md) | Dynamically assign specialized roles to multiple AI agents via a meta-debate protocol (proposal + peer review) before... | 6/10 ★★★ | 182 |
| [evoconfig-self-evolving-multi-agent-systems](skills/evoconfig-self-evolving-multi-agent-systems/SKILL.md) | Autonomous environment configuration using multi-agent diagnosis and self-evolving error repair. Use when: 'set up th... | 6/10 ★★★ | 194 |
| [lemon-agent-technical-report](skills/lemon-agent-technical-report/SKILL.md) | Orchestrate multi-agent workflows using the Lemon Agent orchestrator-worker pattern with hierarchical scheduling, pro... | 6/10 ★★★ | 186 |
| [multi-agent-constraint-factorization-reveals](skills/multi-agent-constraint-factorization-reveals/SKILL.md) | Orchestrate multi-agent LLM pipelines using constraint factorization -- decomposing complex requirements into separat... | 6/10 ★★★ | 157 |
| [understanding-agent-scaling-llm-based](skills/understanding-agent-scaling-llm-based/SKILL.md) | Design diversity-aware multi-agent systems that maximize performance with fewer agents. Uses information-theoretic K*... | 6/10 ★★★ | 204 |
| [why-ai-agents-systematically](skills/why-ai-agents-systematically/SKILL.md) | Diagnose and fix systematic failure modes in LLM-based multi-agent systems performing root cause analysis on cloud in... | 6/10 ★★★ | 309 |
| [agenticpay-multi-agent-negotiation-system](skills/agenticpay-multi-agent-negotiation-system/SKILL.md) | Build multi-agent LLM negotiation systems where buyer and seller agents reach deals through natural language. Use whe... | 5/10 ★★ | 226 |
| [commcp-multi-agent-coordination-llm-based](skills/commcp-multi-agent-coordination-llm-based/SKILL.md) | Build decentralized multi-agent coordination systems using LLM-based communication calibrated with conformal predicti... | 5/10 ★★ | 225 |
| [epistemic-context-learning-building](skills/epistemic-context-learning-building/SKILL.md) | Build trust-aware multi-agent systems using Epistemic Context Learning (ECL). Constructs peer reliability profiles fr... | 5/10 ★★ | 210 |
| [experience-driven-multi-agent-systems-training-fre](skills/experience-driven-multi-agent-systems-training-fre/SKILL.md) | Build self-evolving multi-agent systems that accumulate tool-level expertise through structured interaction without m... | 5/10 ★★ | 168 |
| [flyaoc-evaluating-agentic-ontology](skills/flyaoc-evaluating-agentic-ontology/SKILL.md) | Build multi-agent systems for end-to-end ontology curation from scientific literature. Applies FlyAOC's agent archite... | 5/10 ★★ | 184 |
| [from-assumptions-actions-turning](skills/from-assumptions-actions-turning/SKILL.md) | Build uncertainty-aware planners for multi-agent systems using the PCE (Planner-Composer-Evaluator) decision tree fra... | 5/10 ★★ | 242 |
| [infa-guard-mitigating-malicious-propagation](skills/infa-guard-mitigating-malicious-propagation/SKILL.md) | Implement infection-aware security for LLM multi-agent systems using INFA-Guard's three-category detection (benign/at... | 5/10 ★★ | 262 |
| [llms-as-orchestrators-constraint-compliant](skills/llms-as-orchestrators-constraint-compliant/SKILL.md) | Build constraint-compliant multi-objective recommendation systems using a dual-agent architecture coordinated by an L... | 5/10 ★★ | 186 |
| [mas-prove-understanding-process-verification](skills/mas-prove-understanding-process-verification/SKILL.md) | Design and implement process verification for multi-agent LLM systems. Add intermediate-step evaluation to multi-agen... | 5/10 ★★ | 237 |
| [mascot-multi-agent-socio-collaborative-companion](skills/mascot-multi-agent-socio-collaborative-companion/SKILL.md) | Design and orchestrate multi-agent companion systems where each agent maintains a distinct persona and contributes di... | 5/10 ★★ | 244 |
| [metagen-self-evolving-roles-topologies](skills/metagen-self-evolving-roles-topologies/SKILL.md) | Self-evolving multi-agent orchestration that dynamically generates specialized roles and collaboration topologies at ... | 5/10 ★★ | 189 |
| [multi-agent-causal-reasoning-system](skills/multi-agent-causal-reasoning-system/SKILL.md) | Build multi-agent systems that discover causal rules from event sequences using specialized agents (causal discovery,... | 5/10 ★★ | 225 |
| [multivis-agent-multi-agent-framework-logic](skills/multivis-agent-multi-agent-framework-logic/SKILL.md) | Build reliable multi-agent data visualization pipelines with logic rule constraints. Use when: 'generate a chart from... | 5/10 ★★ | 193 |
| [pamas-self-adaptive-multi-agent-system](skills/pamas-self-adaptive-multi-agent-system/SKILL.md) | Build hierarchical multi-agent systems that detect misinformation, anomalies, and deceptive content using perspective... | 5/10 ★★ | 162 |
| [refer-agent-collaborative-multi-agent-system](skills/refer-agent-collaborative-multi-agent-system/SKILL.md) | Build collaborative multi-agent systems that use alternating reasoning-reflection cycles with specialized agent roles... | 5/10 ★★ | 179 |
| [rulesmith-multi-agent-automated-game](skills/rulesmith-multi-agent-automated-game/SKILL.md) | Automated game balancing using multi-agent LLM self-play coupled with Bayesian optimization. Use when the user asks t... | 5/10 ★★ | 193 |
| [social-catalysts-not-moral](skills/social-catalysts-not-moral/SKILL.md) | Design and audit multi-agent LLM systems for genuine cooperation vs. surface compliance. Implements Anchoring Agent i... | 5/10 ★★ | 180 |
| [status-hierarchies](skills/status-hierarchies/SKILL.md) | Detect and mitigate status hierarchy bias in multi-agent LLM systems. Applies expectation states theory to audit defe... | 5/10 ★★ | 229 |
| [symphony-synergistic-multi-agent-planning](skills/symphony-synergistic-multi-agent-planning/SKILL.md) | Orchestrate heterogeneous multi-agent MCTS planning for complex reasoning and search tasks. Uses a pool of diverse LL... | 5/10 ★★ | 201 |
| [towards-adaptive-scalable-robust](skills/towards-adaptive-scalable-robust/SKILL.md) | Implement RAPS (Reputation-Aware Publish-Subscribe) multi-agent coordination using intent-based pub/sub messaging, re... | 5/10 ★★ | 205 |
| [towards-science-collective-ai](skills/towards-science-collective-ai/SKILL.md) | Design, evaluate, and optimize LLM multi-agent systems using the Collaboration Gain (Gamma) framework. Replaces trial... | 5/10 ★★ | 184 |
| [ts-debate-multimodal-collaborative-debate](skills/ts-debate-multimodal-collaborative-debate/SKILL.md) | Zero-shot time series reasoning via modality-specialized multi-agent debate. Assigns dedicated text, visual, and nume... | 5/10 ★★ | 232 |
| [villain-at-averimatec-verifying](skills/villain-at-averimatec-verifying/SKILL.md) | Build multi-agent fact-checking pipelines that verify image-text claims through modality-specific analysis, cross-mod... | 5/10 ★★ | 248 |
| [when-agents-misremember-collectively](skills/when-agents-misremember-collectively/SKILL.md) | Detect, measure, and defend against collective false-memory propagation (the Mandela Effect) in LLM multi-agent syste... | 5/10 ★★ | 225 |
| [autonomous-multi-agent-ai-high-throughput](skills/autonomous-multi-agent-ai-high-throughput/SKILL.md) | Build multi-agent AI systems for high-throughput scientific workflows with metacognitive self-assessment. Implements ... | 4/10 ★★ | 189 |
| [cam-causality-based-analysis-framework](skills/cam-causality-based-analysis-framework/SKILL.md) | Analyze and optimize multi-agent code generation pipelines using causality-based importance ranking of intermediate f... | 4/10 ★★ | 153 |
| [cowork-x-experience-optimized-co-evolution-multi-a](skills/cowork-x-experience-optimized-co-evolution-multi-a/SKILL.md) | Build multi-agent collaboration systems with experience-driven co-evolution using HTN skill libraries and post-episod... | 4/10 ★★ | 150 |
| [internet-agentic-ai-incentive-compatible](skills/internet-agentic-ai-incentive-compatible/SKILL.md) | Design and implement incentive-compatible multi-agent coalition systems where heterogeneous AI agents dynamically for... | 4/10 ★★ | 208 |
| [moco-one-stop-shop-collaboration](skills/moco-one-stop-shop-collaboration/SKILL.md) | Design and implement multi-LM collaboration pipelines using the MoCo framework's 26 methods across four collaboration... | 4/10 ★★ | 225 |
| [on-uncertainty-model-based-multi-agent](skills/on-uncertainty-model-based-multi-agent/SKILL.md) | Apply entropy-based uncertainty analysis to multi-agent LLM systems. Diagnose when multi-agent setups hurt performanc... | 4/10 ★★ | 184 |
| [thinktank-me-multi-expert-framework-middle](skills/thinktank-me-multi-expert-framework-middle/SKILL.md) | Build multi-expert forecasting systems where specialized LLM agents collaborate through routing and aggregation to pr... | 4/10 ★★ | 207 |

---

## RAG & Retrieval

**42 skills** | Avg rating: 5.6/10

| Skill | Description | Rating | Lines |
|-------|-------------|--------|-------|
| [greprag-empirical-study-optimization](skills/greprag-empirical-study-optimization/SKILL.md) | Lightweight, index-free repository-level code retrieval using ripgrep for context-aware code completion. Uses LLM-gen... | 8/10 ★★★★ | 190 |
| [a-rag-scaling-agentic-retrieval-augmented](skills/a-rag-scaling-agentic-retrieval-augmented/SKILL.md) | Build agentic RAG systems where the LLM autonomously decides retrieval strategy using hierarchical interfaces (keywor... | 7/10 ★★★★ | 253 |
| [aligncoder-aligning-retrieval-target](skills/aligncoder-aligning-retrieval-target/SKILL.md) | Repository-level code completion using AlignCoder's query enhancement and aligned retrieval technique. Generates cand... | 7/10 ★★★★ | 206 |
| [chunking-retrieval-re-ranking-empirical-evaluation](skills/chunking-retrieval-re-ranking-empirical-evaluation/SKILL.md) | Build and optimize two-stage RAG pipelines with bi-encoder retrieval, cross-encoder re-ranking, and empirically-valid... | 7/10 ★★★★ | 230 |
| [comprehensive-comparison-rag-methods](skills/comprehensive-comparison-rag-methods/SKILL.md) | Select and configure the right RAG strategy for conversational QA systems based on dataset characteristics. Use when:... | 7/10 ★★★★ | 167 |
| [deepera-deep-evidence-reranking](skills/deepera-deep-evidence-reranking/SKILL.md) | Rerank retrieved passages for RAG pipelines using step-by-step logical reasoning to filter out semantically similar b... | 7/10 ★★★★ | 223 |
| [towards-ai-evaluation-domain-specific](skills/towards-ai-evaluation-domain-specific/SKILL.md) | Build and evaluate domain-specific RAG systems with iterative user-feedback refinement, source grounding, and structu... | 7/10 ★★★★ | 260 |
| [use-graph-it-needs](skills/use-graph-it-needs/SKILL.md) | Implement adaptive RAG pipelines that route queries to dense retrieval, graph-based retrieval, or a weighted fusion b... | 7/10 ★★★★ | 254 |
| [when-iterative-rag-beats](skills/when-iterative-rag-beats/SKILL.md) | Build iterative retrieval-reasoning RAG pipelines that outperform single-shot retrieval, using staged evidence gather... | 7/10 ★★★★ | 244 |
| [when-should-search-more](skills/when-should-search-more/SKILL.md) | Adaptive complex query optimization for RAG pipelines. Decides when a user query needs decomposition into multiple su... | 7/10 ★★★★ | 300 |
| [a2rag-adaptive-agentic-graph](skills/a2rag-adaptive-agentic-graph/SKILL.md) | Build adaptive, cost-aware Graph-RAG pipelines that route queries through escalating retrieval stages (local -> bridg... | 6/10 ★★★ | 229 |
| [cost-efficient-rag-entity-matching](skills/cost-efficient-rag-entity-matching/SKILL.md) | Build cost-efficient RAG pipelines for entity matching and deduplication using blocking-based batch retrieval and gen... | 6/10 ★★★ | 187 |
| [craft-calibrated-reasoning-answer-faithful](skills/craft-calibrated-reasoning-answer-faithful/SKILL.md) | Apply CRAFT (Calibrated Reasoning with Answer-Faithful Traces) for multi-hop question answering with verified reasoni... | 6/10 ★★★ | 192 |
| [deepread-document-structure-aware-reasoning](skills/deepread-document-structure-aware-reasoning/SKILL.md) | Structure-aware document reasoning that converts PDFs/long documents into hierarchically indexed paragraphs with coor... | 6/10 ★★★ | 187 |
| [diverge-diversity-enhanced-rag-open-ended](skills/diverge-diversity-enhanced-rag-open-ended/SKILL.md) | Diversity-enhanced RAG for open-ended queries with multiple valid answers. Uses reflection-guided generation and memo... | 6/10 ★★★ | 177 |
| [graph-anchored-knowledge-indexing-retrieval-augmen](skills/graph-anchored-knowledge-indexing-retrieval-augmen/SKILL.md) | Build iterative RAG pipelines that construct evolving knowledge graphs to anchor retrieval across multiple hops. Use ... | 6/10 ★★★ | 221 |
| [legalmalr-multi-agent-query-understanding](skills/legalmalr-multi-agent-query-understanding/SKILL.md) | Multi-agent query reformulation and LLM reranking for retrieval over legal, regulatory, or domain-specific corpora. U... | 6/10 ★★★ | 168 |
| [multi-field-tool-retrieval](skills/multi-field-tool-retrieval/SKILL.md) | Implement multi-field tool retrieval systems that decompose tool documentation into structured fields (description, p... | 6/10 ★★★ | 284 |
| [ragturk-best-practices-retrieval](skills/ragturk-best-practices-retrieval/SKILL.md) | Design and optimize RAG pipelines for Turkish and other morphologically rich languages (Turkish, Finnish, Hungarian, ... | 6/10 ★★★ | 208 |
| [research-multi-stage-machine-learning](skills/research-multi-stage-machine-learning/SKILL.md) | Build multi-stage search pipelines that separate recall from precision for discovering datasets, documents, or resour... | 6/10 ★★★ | 279 |
| [sparc-rag-adaptive-sequential-parallel-scaling](skills/sparc-rag-adaptive-sequential-parallel-scaling/SKILL.md) | Implement multi-agent RAG systems with coordinated sequential-parallel scaling and shared context management for comp... | 6/10 ★★★ | 248 |
| [atomic-information-flow-network](skills/atomic-information-flow-network/SKILL.md) | Trace and attribute RAG system responses back to specific tools and sources using Atomic Information Flow (AIF) -- a ... | 5/10 ★★ | 197 |
| [compact-hypercube-embeddings-fast](skills/compact-hypercube-embeddings-fast/SKILL.md) | Build fast similarity-search systems using compact binary hypercube embeddings derived from foundation model encoders... | 5/10 ★★ | 192 |
| [compactrag-reducing-calls-token](skills/compactrag-reducing-calls-token/SKILL.md) | Build multi-hop RAG systems that answer complex questions with only 2 LLM calls total, regardless of reasoning depth.... | 5/10 ★★ | 194 |
| [diffusion-pretrained-dense-contextual-embeddings](skills/diffusion-pretrained-dense-contextual-embeddings/SKILL.md) | Build production retrieval systems using pplx-embed, diffusion-pretrained dense and contextualized embedding models w... | 5/10 ★★ | 183 |
| [dziribot-rag-intelligent-conversational](skills/dziribot-rag-intelligent-conversational/SKILL.md) | Build dialect-aware RAG conversational agents that handle non-standard orthography, code-switching, and multi-script ... | 5/10 ★★ | 236 |
| [evaluating-retrievalaugmented-generation-variants](skills/evaluating-retrievalaugmented-generation-variants/SKILL.md) | Build production-grade natural language to SQL/API pipelines using RAG variant selection (Standard RAG, Self-RAG, CoR... | 5/10 ★★ | 223 |
| [evaluation-oncotimia-system-supporting](skills/evaluation-oncotimia-system-supporting/SKILL.md) | Build RAG pipelines that transform unstructured clinical or domain-specific documents into structured form records us... | 5/10 ★★ | 213 |
| [mermaid-memory-enhanced-retrieval-reasoning](skills/mermaid-memory-enhanced-retrieval-reasoning/SKILL.md) | Memory-enhanced multi-agent retrieval and reasoning for veracity assessment and fact-checking. Use when: 'verify this... | 5/10 ★★ | 189 |
| [mrag-benchmarking-retrieval-augmented-generation](skills/mrag-benchmarking-retrieval-augmented-generation/SKILL.md) | Build and evaluate biomedical RAG pipelines using the MRAG benchmark methodology. Configures retrieval, prompting, an... | 5/10 ★★ | 183 |
| [papersearchqa-learning-search-reason](skills/papersearchqa-learning-search-reason/SKILL.md) | Build iterative search-and-reason agents for scientific literature QA. Uses the PaperSearchQA pattern: interleaved th... | 5/10 ★★ | 233 |
| [pearl-prototype-enhanced-alignment-label-efficient](skills/pearl-prototype-enhanced-alignment-label-efficient/SKILL.md) | Implements PEARL (Prototype-Enhanced Aligned Representation Learning) to improve embedding quality for nearest-neighb... | 5/10 ★★ | 222 |
| [reasoning-augmented-representations-multimodal-ret](skills/reasoning-augmented-representations-multimodal-ret/SKILL.md) | Decouple reasoning from embedding compression in multimodal retrieval pipelines by enriching queries and corpus entri... | 5/10 ★★ | 223 |
| [rethinking-reranker-boundary-aware-evidence](skills/rethinking-reranker-boundary-aware-evidence/SKILL.md) | Implement boundary-aware evidence selection for RAG systems using the BAR-RAG technique. Replaces relevance-only rera... | 5/10 ★★ | 159 |
| [sar-rag-atr-visual-question](skills/sar-rag-atr-visual-question/SKILL.md) | Build image retrieval-augmented generation (ImageRAG) pipelines for visual recognition tasks. Combines vision embeddi... | 5/10 ★★ | 378 |
| [tutorial-reasoning-ir-ir](skills/tutorial-reasoning-ir-ir/SKILL.md) | Build reasoning-enhanced information retrieval pipelines that go beyond semantic matching. Applies five methodologica... | 5/10 ★★ | 249 |
| [what-should-cite-rag](skills/what-should-cite-rag/SKILL.md) | Build multi-level RAG pipelines for academic citation prediction and literature discovery. Use when the user asks to ... | 5/10 ★★ | 193 |
| [deepimagesearch-benchmarking-multimodal-agents](skills/deepimagesearch-benchmarking-multimodal-agents/SKILL.md) | Build agentic image retrieval systems that perform multi-step contextual reasoning over visual histories instead of i... | 4/10 ★★ | 198 |
| [efficient-table-retrieval-understanding](skills/efficient-table-retrieval-understanding/SKILL.md) | Build TabRAG-style pipelines that retrieve relevant tables from large image collections and answer natural language q... | 4/10 ★★ | 178 |
| [sdr-cir-semantic-debias-retrieval](skills/sdr-cir-semantic-debias-retrieval/SKILL.md) | Build training-free composed image retrieval systems that combine a reference image with modification text to find ta... | 4/10 ★★ | 171 |
| [toolweaver-weaving-collaborative-semantics](skills/toolweaver-weaving-collaborative-semantics/SKILL.md) | Design scalable tool retrieval systems using hierarchical code tokenization that captures collaborative tool semantic... | 4/10 ★★ | 181 |
| [viola-video-in-context-learning](skills/viola-video-in-context-learning/SKILL.md) | Apply the VIOLA framework for label-efficient in-context learning on video or multimodal data. Uses density-uncertain... | 4/10 ★★ | 221 |

---

## Efficiency & Optimization

**34 skills** | Avg rating: 5.4/10

| Skill | Description | Rating | Lines |
|-------|-------------|--------|-------|
| [on-impact-agentsmd-files](skills/on-impact-agentsmd-files/SKILL.md) | Generate and optimize AGENTS.md / CLAUDE.md repository instruction files to reduce AI coding agent runtime and token ... | 9/10 ★★★★ | 196 |
| [dr-kernel-reinforcement-learning-done](skills/dr-kernel-reinforcement-learning-done/SKILL.md) | Write high-performance Triton GPU kernels using Dr. Kernel's multi-turn refinement strategy: profile-guided optimizat... | 7/10 ★★★★ | 200 |
| [large-model-powered-evolutionary-code](skills/large-model-powered-evolutionary-code/SKILL.md) | Iteratively optimize code performance using LLM-driven evolutionary search on a phylogenetic tree. Applies PhyloEvolv... | 7/10 ★★★★ | 186 |
| [logsieve-task-aware-ci-log](skills/logsieve-task-aware-ci-log/SKILL.md) | Reduce verbose CI/CD build logs before LLM analysis using RCA-aware semantic filtering. Removes boilerplate lines (de... | 7/10 ★★★★ | 167 |
| [ruleflow-generating-reusable-program](skills/ruleflow-generating-reusable-program/SKILL.md) | Optimize Pandas code by discovering per-program improvements, generalizing them into reusable rewrite rules, and appl... | 7/10 ★★★★ | 160 |
| [semanticalli-caching-reasoning-not](skills/semanticalli-caching-reasoning-not/SKILL.md) | Implement pipeline-aware intermediate representation (IR) caching for agentic systems. Instead of caching final LLM r... | 7/10 ★★★★ | 202 |
| [swe-pruner-self-adaptive-context-pruning](skills/swe-pruner-self-adaptive-context-pruning/SKILL.md) | Apply SWE-Pruner's goal-conditioned context pruning to reduce token usage when working with large codebases. Teaches ... | 7/10 ★★★★ | 186 |
| [when-much-imagine-adaptive](skills/when-much-imagine-adaptive/SKILL.md) | Adaptive test-time scaling framework that decides WHEN and HOW MUCH to invoke expensive generative steps (world model... | 7/10 ★★★★ | 228 |
| [agentcgroup-understanding-controlling-os](skills/agentcgroup-understanding-controlling-os/SKILL.md) | Design and implement OS-level resource controls for sandboxed AI agents using hierarchical cgroups, eBPF enforcement,... | 6/10 ★★★ | 231 |
| [deltaevolve-accelerating-scientific-discovery](skills/deltaevolve-accelerating-scientific-discovery/SKILL.md) | Iteratively evolve code solutions using momentum-driven semantic deltas instead of full-code histories. Use when: 'ev... | 6/10 ★★★ | 187 |
| [markovscale-optimal-sequential-scaling](skills/markovscale-optimal-sequential-scaling/SKILL.md) | Implement MarkovScale's principled sequential scaling for LLM inference pipelines. Models retry/refinement loops as a... | 6/10 ★★★ | 193 |
| [timely-machine-awareness-time](skills/timely-machine-awareness-time/SKILL.md) | Apply time-budget-aware reasoning to agentic tasks with tool calls. Dynamically adjust strategy depth, tool call freq... | 6/10 ★★★ | 171 |
| [towards-automated-kernel-generation](skills/towards-automated-kernel-generation/SKILL.md) | Automate GPU kernel generation and optimization using LLM-driven agentic workflows with profiling feedback loops. Use... | 6/10 ★★★ | 159 |
| [codeocr-effectiveness-vision-code](skills/codeocr-effectiveness-vision-code/SKILL.md) | Render source code as images for vision LLM processing to reduce token cost while preserving understanding. Use when:... | 5/10 ★★ | 241 |
| [contextevolve-multi-agent-context-compression](skills/contextevolve-multi-agent-context-compression/SKILL.md) | Multi-agent iterative code optimization using context compression. Decomposes optimization into three agents (Summari... | 5/10 ★★ | 175 |
| [do-truly-benefit-longer](skills/do-truly-benefit-longer/SKILL.md) | Optimize LLM context length for post-editing and refinement pipelines. Applies research showing that naively adding d... | 5/10 ★★ | 252 |
| [ecco-evidence-driven-causal-reasoning](skills/ecco-evidence-driven-causal-reasoning/SKILL.md) | Apply evidence-driven causal reasoning to compiler optimization pass selection and ordering. Uses the ECCO framework:... | 5/10 ★★ | 205 |
| [fine-tuning-gpt-5-gpu-kernel](skills/fine-tuning-gpt-5-gpu-kernel/SKILL.md) | Generate optimized GPU kernels in Triton from PyTorch reference code using the Makora RL-based iterative refinement w... | 5/10 ★★ | 259 |
| [hqp-sensitivity-aware-hybrid-quantization](skills/hqp-sensitivity-aware-hybrid-quantization/SKILL.md) | Apply the HQP framework to compress and accelerate PyTorch models for edge deployment using sensitivity-aware structu... | 5/10 ★★ | 189 |
| [mmr-bench-comprehensive-benchmark-multimodal](skills/mmr-bench-comprehensive-benchmark-multimodal/SKILL.md) | Build cost-aware multimodal LLM routing systems that select the best model per query based on input signals, budget c... | 5/10 ★★ | 175 |
| [predicting-improving-test-time-scaling](skills/predicting-improving-test-time-scaling/SKILL.md) | Implement Scaling-Law Guided (SLG) Search for test-time compute optimization. Uses reward tail distribution estimatio... | 5/10 ★★ | 209 |
| [profinfer-ebpf-based-fine-grained-inference](skills/profinfer-ebpf-based-fine-grained-inference/SKILL.md) | Profile and diagnose LLM inference engines (llama.cpp and similar GGML-based runtimes) using eBPF uprobes for non-int... | 5/10 ★★ | 289 |
| [tokenomics-quantifying-where-tokens](skills/tokenomics-quantifying-where-tokens/SKILL.md) | Analyze and optimize token consumption in LLM-based multi-agent software engineering workflows. Maps agent execution ... | 5/10 ★★ | 227 |
| [towards-green-ai-decoding](skills/towards-green-ai-decoding/SKILL.md) | Optimize LLM-generated code for energy efficiency by detecting and suppressing babbling behavior (excess tokens like ... | 5/10 ★★ | 229 |
| [trust-design-skill-profiles](skills/trust-design-skill-profiles/SKILL.md) | Budget-aware LLM model selection using BELLA-style skill profiling. Decomposes tasks into granular skill requirements... | 5/10 ★★ | 215 |
| [dart-diffusion-inspired-speculative-decoding](skills/dart-diffusion-inspired-speculative-decoding/SKILL.md) | Set up and use DART (Diffusion-Inspired Speculative Decoding) for fast LLM inference. DART replaces autoregressive dr... | 4/10 ★★ | 241 |
| [hyperoffload-graph-driven-hierarchical-memory](skills/hyperoffload-graph-driven-hierarchical-memory/SKILL.md) | Design and implement compiler-driven hierarchical memory offloading for LLM inference and training on multi-tier memo... | 4/10 ★★ | 226 |
| [more-than-quick-glance](skills/more-than-quick-glance/SKILL.md) | Implement LASER-KV-style KV-cache compression for LLM inference pipelines using block-wise accumulative budgeting and... | 4/10 ★★ | 256 |
| [protean-compiler-agile-framework](skills/protean-compiler-agile-framework/SKILL.md) | Guide fine-grained LLVM compiler phase ordering using the Protean framework's agile optimization approach — clusterin... | 4/10 ★★ | 149 |
| [sere-similarity-based-expert-re-routing](skills/sere-similarity-based-expert-re-routing/SKILL.md) | Deploy SERE (Similarity-based Expert Re-routing) to accelerate MoE model batch decoding in vLLM by dynamically skippi... | 4/10 ★★ | 223 |
| [swe-replay-test-time-scaling-software](skills/swe-replay-test-time-scaling-software/SKILL.md) | Efficient test-time scaling for software engineering agents using trajectory recycling and explore-exploit branching ... | 4/10 ★★ | 158 |
| [unicomp-unified-evaluation-compression](skills/unicomp-unified-evaluation-compression/SKILL.md) | Guide Claude through evaluating and recommending LLM compression strategies (pruning, quantization, distillation) usi... | 4/10 ★★ | 177 |
| [visiontrim-unified-vision-token](skills/visiontrim-unified-vision-token/SKILL.md) | Implement VisionTrim's training-free visual token compression for multimodal LLMs. Combines attention-based dominant ... | 4/10 ★★ | 212 |
| [vtc-r1-vision-text-compression-long-context](skills/vtc-r1-vision-text-compression-long-context/SKILL.md) | Implement VTC-R1 vision-text compression for efficient long-context reasoning. Renders intermediate reasoning segment... | 4/10 ★★ | 269 |

---

## Data Processing

**28 skills** | Avg rating: 6.0/10

| Skill | Description | Rating | Lines |
|-------|-------------|--------|-------|
| [can-clean-up-mess](skills/can-clean-up-mess/SKILL.md) | LLM-driven data preparation pipeline for cleaning, integrating, and enriching messy datasets. Use when the user says ... | 8/10 ★★★★ | 164 |
| [sql-trail-multi-turn-reinforcement-learning](skills/sql-trail-multi-turn-reinforcement-learning/SKILL.md) | Iterative multi-turn Text-to-SQL generation using reason-execute-observe loops with execution feedback. Instead of wr... | 8/10 ★★★★ | 186 |
| [autonomous-data-processing-meta-agents](skills/autonomous-data-processing-meta-agents/SKILL.md) | Build self-managing data processing pipelines using hierarchical meta-agent orchestration. Decomposes complex data ta... | 7/10 ★★★★ | 216 |
| [benchmarking-text-to-python-against-text-to-sql](skills/benchmarking-text-to-python-against-text-to-sql/SKILL.md) | Generate correct Python/Pandas code from natural language questions over tabular data, applying the Logic Completion ... | 7/10 ★★★★ | 188 |
| [corpusqa-10-million-token](skills/corpusqa-10-million-token/SKILL.md) | Corpus-level QA over massive document collections using memory-augmented agentic processing. Synthesize answers that ... | 7/10 ★★★★ | 179 |
| [hunt-instead-wait-evaluating](skills/hunt-instead-wait-evaluating/SKILL.md) | Autonomously explore databases and datasets to extract key insights without predefined queries, using investigatory i... | 7/10 ★★★★ | 222 |
| [sqlagent-learning-explore-before](skills/sqlagent-learning-explore-before/SKILL.md) | Explore unfamiliar databases before writing SQL by building a local knowledge base of schema fragments, executable qu... | 7/10 ★★★★ | 201 |
| [system-name-address-parsing](skills/system-name-address-parsing/SKILL.md) | Parse unstructured person names and addresses into a structured 17-field schema using prompt-driven extraction with l... | 7/10 ★★★★ | 207 |
| [accelerating-social-science-research](skills/accelerating-social-science-research/SKILL.md) | Implement the EXPERIGEN agentic framework for automated hypothesis generation and empirical validation on datasets. U... | 6/10 ★★★ | 177 |
| [aiano-enhancing-information-retrieval](skills/aiano-enhancing-information-retrieval/SKILL.md) | Build AI-augmented annotation pipelines for creating high-quality information retrieval and QA datasets. Combines LLM... | 6/10 ★★★ | 161 |
| [datacross-unified-benchmark-agent](skills/datacross-unified-benchmark-agent/SKILL.md) | Cross-modal data analysis agent that unifies structured sources (SQL, CSV, JSON) with unstructured visual documents (... | 6/10 ★★★ | 203 |
| [harnessing-precision-querying-retrieval-augmented](skills/harnessing-precision-querying-retrieval-augmented/SKILL.md) | LLM-driven precision querying of structured tabular data via Python/Pandas code generation and retrieval-augmented ex... | 6/10 ★★★ | 163 |
| [llm-assisted-logic-rule-learning](skills/llm-assisted-logic-rule-learning/SKILL.md) | Build deterministic, interpretable anomaly detection rule sets for time series data using LLM-driven labeling, symbol... | 6/10 ★★★ | 181 |
| [mata-multiagent-framework-for](skills/mata-multiagent-framework-for/SKILL.md) | Multi-agent table question answering using MATA's three-path reasoning strategy (Chain-of-Thought, Program-of-Thought... | 6/10 ★★★ | 167 |
| [orthogonal-hierarchical-decomposition-structure-aw](skills/orthogonal-hierarchical-decomposition-structure-aw/SKILL.md) | Decompose complex tables with multi-level headers, merged cells, and irregular layouts into orthogonal column/row tre... | 6/10 ★★★ | 231 |
| [refuge-feature-generation-prediction](skills/refuge-feature-generation-prediction/SKILL.md) | Automated feature engineering for prediction tasks on relational databases using a multi-agent LLM pipeline. Generate... | 6/10 ★★★ | 168 |
| [scidatacopilot-agentic-data-preparation](skills/scidatacopilot-agentic-data-preparation/SKILL.md) | Build agentic pipelines that ingest heterogeneous raw scientific data, parse research intent, and produce analysis-re... | 6/10 ★★★ | 240 |
| [st-raptor-agentic-system-semi-structured](skills/st-raptor-agentic-system-semi-structured/SKILL.md) | Agentic system for answering questions about semi-structured tables using tree-based structural modeling and multi-st... | 6/10 ★★★ | 222 |
| [state-art-llm-enabled-interaction](skills/state-art-llm-enabled-interaction/SKILL.md) | Build LLM-powered natural language interfaces for data visualization — NL2VIS pipelines, conversational chart analyti... | 6/10 ★★★ | 258 |
| [krone-hierarchical-modular-log](skills/krone-hierarchical-modular-log/SKILL.md) | Detect anomalies in application logs using KRONE's hierarchical decomposition: parse flat log sequences into Entity/A... | 5/10 ★★ | 240 |
| [large-scale-multidimensional-knowledge-profiling](skills/large-scale-multidimensional-knowledge-profiling/SKILL.md) | Build multidimensional profiling pipelines for large scientific paper corpora. Combines BERTopic clustering, LLM-stru... | 5/10 ★★ | 246 |
| [on-use-support-conduction](skills/on-use-support-conduction/SKILL.md) | LLM-assisted systematic literature review and mapping study pipeline. Automates screening, data extraction, and class... | 5/10 ★★ | 167 |
| [small-beautiful-practical-log](skills/small-beautiful-practical-log/SKILL.md) | Build efficient log parsing systems that extract structured templates from raw log messages using a dual-cache archit... | 5/10 ★★ | 191 |
| [treetensor-boost-ai-system](skills/treetensor-boost-ai-system/SKILL.md) | Implement TreeTensor-based nested data handling for AI systems using the DI-treetensor library. Replaces manual recur... | 5/10 ★★ | 237 |
| [tsaqa-time-series-analysis](skills/tsaqa-time-series-analysis/SKILL.md) | Structured time series question answering using the TSAQA six-task framework: anomaly detection, classification, char... | 5/10 ★★ | 195 |
| [when-meets-fuzzy-topsis-personnel](skills/when-meets-fuzzy-topsis-personnel/SKILL.md) | Rank and select candidates using LLM-scored profiles combined with Fuzzy TOPSIS multi-criteria decision-making. Use w... | 5/10 ★★ | 233 |
| [discovering-high-level-patterns](skills/discovering-high-level-patterns/SKILL.md) | Extract high-level semantic patterns from fine-grained simulation or event logs using LM-guided program synthesis. Tr... | 4/10 ★★ | 190 |
| [realistic-synthetic-household-data](skills/realistic-synthetic-household-data/SKILL.md) | Generate realistic synthetic household datasets with bidirectional persona-environment coupling for embodied AI train... | 4/10 ★★ | 179 |

---

## Prompt Engineering

**27 skills** | Avg rating: 5.8/10

| Skill | Description | Rating | Lines |
|-------|-------------|--------|-------|
| [c-mop-integrating-momentum-boundary-aware](skills/c-mop-integrating-momentum-boundary-aware/SKILL.md) | Optimize LLM system prompts iteratively using boundary-aware contrastive sampling and momentum-guided clustering from... | 7/10 ★★★★ | 179 |
| [discoverllm-executing-intents-discovering](skills/discoverllm-executing-intents-discovering/SKILL.md) | Help users discover and form their intents through adaptive diverge-converge interaction, rather than just asking cla... | 7/10 ★★★★ | 227 |
| [error-taxonomy-guided-prompt-optimization](skills/error-taxonomy-guided-prompt-optimization/SKILL.md) | Optimize LLM prompts by systematically collecting errors, building a taxonomy of failure modes, and augmenting prompt... | 7/10 ★★★★ | 162 |
| [gflowpo-generative-flow-network](skills/gflowpo-generative-flow-network/SKILL.md) | Optimize LLM prompts using GFlowPO's iterative generate-evaluate-refine loop with diversity-preserving exploration an... | 7/10 ★★★★ | 171 |
| [llm-based-sql-generation-prompting](skills/llm-based-sql-generation-prompting/SKILL.md) | Generate accurate SQL from natural language using the SSEV pipeline: schema-linked prompting, execution-guided self-r... | 7/10 ★★★★ | 174 |
| [reprompt-prompt-generation-intelligent](skills/reprompt-prompt-generation-intelligent/SKILL.md) | Generate optimized system and user prompts for coding agents using requirements engineering principles from the REpro... | 7/10 ★★★★ | 191 |
| [think-augmented-function-calling-improving](skills/think-augmented-function-calling-improving/SKILL.md) | Improve LLM function/tool calling accuracy by injecting explicit "think" reasoning parameters into function schemas b... | 7/10 ★★★★ | 262 |
| [thinking-makes-agents-introverted](skills/thinking-makes-agents-introverted/SKILL.md) | Prevents the "introverted agent" problem where extended reasoning causes agents to give shorter, less informative res... | 7/10 ★★★★ | 244 |
| [dancing-chains-strategic-persuasion](skills/dancing-chains-strategic-persuasion/SKILL.md) | Apply Theory of Mind-based strategic persuasion to code reviews, PR rebuttals, RFC objections, and technical disagree... | 6/10 ★★★ | 266 |
| [do-reasoning-ask-questions](skills/do-reasoning-ask-questions/SKILL.md) | Information-theoretic question-asking framework for disambiguating user intent through structured yes/no questions. U... | 6/10 ★★★ | 183 |
| [funny-or-persuasive-but](skills/funny-or-persuasive-but/SKILL.md) | Fine-grained multi-concept text control that avoids the compositionality trap where LLMs degrade when asked to be e.g... | 6/10 ★★★ | 178 |
| [how-few-shot-demonstrations-affect](skills/how-few-shot-demonstrations-affect/SKILL.md) | Design prompt-based LLM safety defenses using optimal few-shot strategies. Applies the finding that few-shot demonstr... | 6/10 ★★★ | 199 |
| [prompt-driven-development-claude](skills/prompt-driven-development-claude/SKILL.md) | Orchestrate prompt-driven development of large multi-module systems through structured, iterative natural-language wo... | 6/10 ★★★ | 188 |
| [reflect-transparent-principle-guided-reasoning](skills/reflect-transparent-principle-guided-reasoning/SKILL.md) | Apply the REFLECT constitutional alignment framework to enforce user-defined principles on LLM outputs through a mult... | 6/10 ★★★ | 185 |
| [tracellm-leveraging-prompt-engineering](skills/tracellm-leveraging-prompt-engineering/SKILL.md) | Establish and verify traceability links between software artifacts (requirements, design docs, test cases, regulation... | 6/10 ★★★ | 179 |
| [controlling-output-rankings-generative](skills/controlling-output-rankings-generative/SKILL.md) | Optimize product/content descriptions to influence rankings in LLM-based search engines (generative engines) using th... | 5/10 ★★ | 245 |
| [darl-encouraging-diverse-answers](skills/darl-encouraging-diverse-answers/SKILL.md) | Generate diverse, high-quality answer variants for open-ended tasks using DARL's bounded-diversity framework. Use whe... | 5/10 ★★ | 286 |
| [interpreting-controlling-behavior-constitutions](skills/interpreting-controlling-behavior-constitutions/SKILL.md) | Learn and apply natural-language constitutions that map prompt edits to predictable model behavior changes. Use atomi... | 5/10 ★★ | 182 |
| [knowledge-restoration-driven-prompt-optimization](skills/knowledge-restoration-driven-prompt-optimization/SKILL.md) | Iteratively optimize LLM prompts for information extraction tasks using self-evaluation feedback loops. Applies the K... | 5/10 ★★ | 211 |
| [less-noise-more-voice](skills/less-noise-more-voice/SKILL.md) | Identify and remove interference tokens from prompts to improve LLM reasoning accuracy. Based on the LENS framework (... | 5/10 ★★ | 236 |
| [lhaw-controllable-underspecification-long-horizon](skills/lhaw-controllable-underspecification-long-horizon/SKILL.md) | Detect and handle ambiguity in long-horizon agent tasks using the LHAW framework. Systematically identify underspecif... | 5/10 ★★ | 170 |
| [llm-prompt-evaluation-educational](skills/llm-prompt-evaluation-educational/SKILL.md) | Systematically design, evaluate, and rank LLM prompts for educational applications using tournament-style Glicko-2 co... | 5/10 ★★ | 223 |
| [personality-as-relational-infrastructure](skills/personality-as-relational-infrastructure/SKILL.md) | Design LLM messaging systems that infuse Big Five personality traits for sustained user engagement. Uses aggregate-ex... | 5/10 ★★ | 252 |
| [probing-knowledge-boundary-interactive](skills/probing-knowledge-boundary-interactive/SKILL.md) | Systematically extract deep knowledge from LLMs using an interactive agentic framework with four adaptive exploration... | 5/10 ★★ | 181 |
| [textual-equilibrium-propagation-deep](skills/textual-equilibrium-propagation-deep/SKILL.md) | Optimize deep multi-step AI pipelines using Textual Equilibrium Propagation (TEP) — a two-phase local-then-nudge stra... | 5/10 ★★ | 147 |
| [usage-effects-requirements-ai-coding](skills/usage-effects-requirements-ai-coding/SKILL.md) | Optimize AI coding assistant interactions using empirical enterprise findings on usage patterns, productivity factors... | 5/10 ★★ | 228 |
| [culturally-grounded-personas-characterization](skills/culturally-grounded-personas-characterization/SKILL.md) | Generate and evaluate culturally-grounded LLM personas using World Values Survey variables, Inglehart-Welzel Cultural... | 4/10 ★★ | 202 |

---

## NLP & Text

**24 skills** | Avg rating: 4.9/10

| Skill | Description | Rating | Lines |
|-------|-------------|--------|-------|
| [agentcpm-report-interleaving-drafting-deepening](skills/agentcpm-report-interleaving-drafting-deepening/SKILL.md) | Generate deep research reports by interleaving evidence-based drafting with reasoning-driven deepening. Uses the WARP... | 6/10 ★★★ | 232 |
| [drpg-decompose-retrieve-plan](skills/drpg-decompose-retrieve-plan/SKILL.md) | Structured rebuttal and critique-response generation using the DRPG framework (Decompose, Retrieve, Plan, Generate). ... | 6/10 ★★★ | 253 |
| [linguistagent-a-reflective-multimodel](skills/linguistagent-a-reflective-multimodel/SKILL.md) | Implements a reflective dual-agent (Annotator + Reviewer) workflow for automated linguistic annotation tasks such as ... | 6/10 ★★★ | 241 |
| [adaptbpe-general-purpose-specialized](skills/adaptbpe-general-purpose-specialized/SKILL.md) | Adapt general-purpose BPE tokenizers into domain- or language-specialized tokenizers using the AdaptBPE post-training... | 5/10 ★★ | 202 |
| [are-open-weight-ready-social](skills/are-open-weight-ready-social/SKILL.md) | Build LLM-based content moderation pipelines using zero-shot classification with open-weight models. Implements the s... | 5/10 ★★ | 264 |
| [assessment-generative-named-entity](skills/assessment-generative-named-entity/SKILL.md) | Build generative NER systems using LLMs with optimal output formats and prompt engineering. Use when: 'extract entiti... | 5/10 ★★ | 245 |
| [benchmarking-pairwise-causal-discovery](skills/benchmarking-pairwise-causal-discovery/SKILL.md) | Detect and extract pairwise causal relationships from text using structured prompting strategies (zero-shot, CoT, few... | 5/10 ★★ | 204 |
| [beyond-holistic-scores-automatic](skills/beyond-holistic-scores-automatic/SKILL.md) | Build trait-based essay scoring systems that evaluate argumentative writing across multiple rubric dimensions (Conten... | 5/10 ★★ | 201 |
| [cognitively-diverse-multiple-choice-question](skills/cognitively-diverse-multiple-choice-question/SKILL.md) | Generate high-quality multiple-choice questions at controlled cognitive levels using the ReQUESTA multi-agent framewo... | 5/10 ★★ | 177 |
| [cost-aware-selection-text-classification](skills/cost-aware-selection-text-classification/SKILL.md) | Guides cost-aware model selection for text classification pipelines, applying multi-objective trade-off analysis (F1 ... | 5/10 ★★ | 174 |
| [fmbench-adaptive-output-formatting](skills/fmbench-adaptive-output-formatting/SKILL.md) | Adaptive Markdown output formatting that balances semantic fidelity with structural correctness. Applies the FMBench ... | 5/10 ★★ | 224 |
| [human-aligned-enhancement-programming-answers](skills/human-aligned-enhancement-programming-answers/SKILL.md) | Enhance programming answers by classifying user feedback comments as actionable or non-actionable, then surgically in... | 5/10 ★★ | 258 |
| [large-geolocation-extraction-humanitarian](skills/large-geolocation-extraction-humanitarian/SKILL.md) | Extract and geocode location mentions from humanitarian and crisis texts using a two-step LLM pipeline: few-shot NER ... | 5/10 ★★ | 213 |
| [lata-tool-llm-assisted-translation](skills/lata-tool-llm-assisted-translation/SKILL.md) | LLM-assisted translation annotation: build parallel corpus annotation pipelines with template-based prompt management... | 5/10 ★★ | 232 |
| [leveraging-turkish-skill-extraction](skills/leveraging-turkish-skill-extraction/SKILL.md) | Extract and normalize skills from job postings using a two-stage LLM pipeline: dynamic few-shot skill identification ... | 5/10 ★★ | 198 |
| [optimizing-small-sample-experience-learning-llm-ba](skills/optimizing-small-sample-experience-learning-llm-ba/SKILL.md) | Implement the ExperienceWeaver hierarchical experience-learning framework to improve text quality from small feedback... | 5/10 ★★ | 195 |
| [standardizing-longitudinal-radiology-report](skills/standardizing-longitudinal-radiology-report/SKILL.md) | Build LLM-based pipelines that automatically detect and classify longitudinal (temporal) changes in radiology reports... | 5/10 ★★ | 230 |
| [text-summarization-global-structure](skills/text-summarization-global-structure/SKILL.md) | Summarize long documents while preserving global semantic structure and logical coherence using topology-guided pruni... | 5/10 ★★ | 165 |
| [birdturk-adaptation-bird-text-to-sql](skills/birdturk-adaptation-bird-text-to-sql/SKILL.md) | Adapt Text-to-SQL systems and benchmarks for non-English, morphologically rich languages using controlled translation... | 4/10 ★★ | 242 |
| [can-small-handle-context-summarized](skills/can-small-handle-context-summarized/SKILL.md) | Build context-summarized multi-turn QA systems that let small language models (SLMs) handle customer-service dialogue... | 4/10 ★★ | 254 |
| [conceptual-cultural-index-metric](skills/conceptual-cultural-index-metric/SKILL.md) | Compute the Conceptual Cultural Index (CCI) to measure cultural specificity of sentences using LLM-based generality e... | 4/10 ★★ | 246 |
| [hybrid-supervised-llm-pipeline-actionable-suggesti](skills/hybrid-supervised-llm-pipeline-actionable-suggesti/SKILL.md) | Build hybrid classifier-then-LLM pipelines to extract actionable suggestions from unstructured customer reviews. Use ... | 4/10 ★★ | 193 |
| [optimal-turkish-subword-strategies](skills/optimal-turkish-subword-strategies/SKILL.md) | Design and evaluate subword tokenizers for Turkish and other morphologically rich languages (MRLs) using the vocabula... | 4/10 ★★ | 253 |
| [sogptspotter-detecting-chatgpt-generated-answers](skills/sogptspotter-detecting-chatgpt-generated-answers/SKILL.md) | Detect AI-generated answers in Q&A content using Siamese embedding comparison with reference-answer anchoring. Trigge... | 4/10 ★★ | 200 |

---

## Multimodal

**21 skills** | Avg rating: 5.0/10

| Skill | Description | Rating | Lines |
|-------|-------------|--------|-------|
| [chart-specification-structural-representations](skills/chart-specification-structural-representations/SKILL.md) | Generate high-fidelity plotting code from chart images or descriptions using structured intermediate specifications. ... | 7/10 ★★★★ | 344 |
| [svrepair-structured-visual-reasoning](skills/svrepair-structured-visual-reasoning/SKILL.md) | Fix bugs using structured visual reasoning -- converts screenshots, control-flow graphs, and UI artifacts into semant... | 7/10 ★★★★ | 193 |
| [chatting-images-introspective-visual](skills/chatting-images-introspective-visual/SKILL.md) | Apply introspective visual thinking by iteratively 'chatting with images' — using language-guided re-examination of v... | 6/10 ★★★ | 186 |
| [code2world-gui-world-renderable](skills/code2world-gui-world-renderable/SKILL.md) | Predict and simulate GUI state transitions by generating renderable HTML/CSS/SVG code from screenshots and user actio... | 6/10 ★★★ | 292 |
| [decoupling-skeleton-flesh-multimodal](skills/decoupling-skeleton-flesh-multimodal/SKILL.md) | Disentangled structure-content reasoning for table images and structured data. Separates table skeleton (layout/struc... | 6/10 ★★★ | 180 |
| [sparc-separating-perception-reasoning](skills/sparc-separating-perception-reasoning/SKILL.md) | Decouple visual perception from reasoning when building VLM pipelines, image analysis agents, or multi-modal workflow... | 6/10 ★★★ | 170 |
| [vistira-closing-image-text-modality](skills/vistira-closing-image-text-modality/SKILL.md) | Solve math problems from images by decomposing them into interleaved natural-language rationales and executable Pytho... | 6/10 ★★★ | 189 |
| [calliope-tts-based-narrated-e-book](skills/calliope-tts-based-narrated-e-book/SKILL.md) | Build offline TTS-narrated e-books with exact audio-text synchronization in EPUB 3 Media Overlay format. Use when the... | 5/10 ★★ | 178 |
| [gdcnet-generative-discrepancy-comparison](skills/gdcnet-generative-discrepancy-comparison/SKILL.md) | Detect sarcasm and semantic incongruity in multimodal (image+text) content using the GDCNet three-channel discrepancy... | 5/10 ★★ | 180 |
| [raicl-retrieval-augmented-in-context-learning](skills/raicl-retrieval-augmented-in-context-learning/SKILL.md) | Build retrieval-augmented in-context learning (RAICL) pipelines that convert time-series or signal data into images a... | 5/10 ★★ | 228 |
| [unikie-bench-benchmarking-multimodal-key](skills/unikie-bench-benchmarking-multimodal-key/SKILL.md) | Extract structured key information from document images using schema-guided prompting for LMMs. Builds KIE pipelines ... | 5/10 ★★ | 292 |
| [vision-deepresearch-incentivizing-deepresearch-cap](skills/vision-deepresearch-incentivizing-deepresearch-cap/SKILL.md) | Multi-turn, multi-entity, multi-scale visual and textual deep research agent for answering complex questions about im... | 5/10 ★★ | 180 |
| [zero-shot-product-attribute-labeling](skills/zero-shot-product-attribute-labeling/SKILL.md) | Extract and classify product attributes from images using Vision-Language Models with structured prompts and a three-... | 5/10 ★★ | 268 |
| [beyond-translation-cross-cultural-meme](skills/beyond-translation-cross-cultural-meme/SKILL.md) | Cross-cultural meme transcreation using a three-stage hybrid pipeline (cultural analysis, visual generation, assembly... | 4/10 ★★ | 170 |
| [computational-approach-visual-metonymy](skills/computational-approach-visual-metonymy/SKILL.md) | Generate and evaluate visual metonymy -- indirect visual representations that evoke concepts through associated cues ... | 4/10 ★★ | 179 |
| [event-vstream-event-driven-real-time-understanding](skills/event-vstream-event-driven-real-time-understanding/SKILL.md) | Build event-driven video stream processing pipelines that detect meaningful state transitions instead of processing e... | 4/10 ★★ | 254 |
| [gutenocr-grounded-vision-language-front-end](skills/gutenocr-grounded-vision-language-front-end/SKILL.md) | Build grounded OCR pipelines using GutenOCR's prompt-based interface for reading, detection, and spatial grounding on... | 4/10 ★★ | 210 |
| [leveraging-data-say-no](skills/leveraging-data-say-no/SKILL.md) | Implement memory-augmented selective prediction for vision-language models using retrieval-based confidence scoring a... | 4/10 ★★ | 194 |
| [modality-gap-driven-subspace-alignment](skills/modality-gap-driven-subspace-alignment/SKILL.md) | Align multimodal embeddings (vision-language) by correcting the modality gap using the ReAlign/ReVision technique. Fi... | 4/10 ★★ | 231 |
| [paperbanana-automating-academic-illustration](skills/paperbanana-automating-academic-illustration/SKILL.md) | Generate publication-ready academic illustrations using a multi-agent pipeline inspired by PaperBanana. Orchestrates ... | 4/10 ★★ | 197 |
| [vista-scene-aware-optimization-streaming](skills/vista-scene-aware-optimization-streaming/SKILL.md) | Implement Vista-style scene-aware streaming video processing pipelines with dynamic segmentation, hierarchical compre... | 4/10 ★★ | 261 |

---

## Memory & Context

**20 skills** | Avg rating: 5.6/10

| Skill | Description | Rating | Lines |
|-------|-------------|--------|-------|
| [agentsm-semantic-memory-agentic](skills/agentsm-semantic-memory-agentic/SKILL.md) | Agentic Text-to-SQL with semantic memory that captures and reuses structured execution traces. Use when: 'write SQL f... | 7/10 ★★★★ | 213 |
| [fademem-biologically-inspired-forgetting-agent](skills/fademem-biologically-inspired-forgetting-agent/SKILL.md) | Implement biologically-inspired forgetting mechanisms for LLM agent memory systems. Build dual-layer memory hierarchi... | 7/10 ★★★★ | 222 |
| [loca-bench-benchmarking-agents-under](skills/loca-bench-benchmarking-agents-under/SKILL.md) | Apply context management strategies from LOCA-bench to prevent context rot in long-running agent tasks. Implements pr... | 7/10 ★★★★ | 169 |
| [structured-context-engineering-file-native](skills/structured-context-engineering-file-native/SKILL.md) | Structure database schemas and structured data as file-native context for LLM agent operations. Applies evidence-base... | 7/10 ★★★★ | 302 |
| [ama-adaptive-memory-multi-agent](skills/ama-adaptive-memory-multi-agent/SKILL.md) | Build adaptive memory systems using coordinated multi-agent collaboration with hierarchical storage and consistency m... | 6/10 ★★★ | 224 |
| [harmoni-multimodal-personalization-multi-user](skills/harmoni-multimodal-personalization-multi-user/SKILL.md) | Build multi-user personalization pipelines with per-user profile tracking, multimodal perception, and LLM-driven cont... | 6/10 ★★★ | 197 |
| [live-evo-online-evolution-agentic](skills/live-evo-online-evolution-agentic/SKILL.md) | Implement online self-evolving memory for LLM agents using dual-bank architecture (Experience Bank + Meta-Guideline B... | 6/10 ★★★ | 204 |
| [swe-context-bench-benchmark](skills/swe-context-bench-benchmark/SKILL.md) | Reuse prior coding experience across related repository tasks. Accumulate, summarize, retrieve, and inject compact ex... | 6/10 ★★★ | 180 |
| [tracemem-weaving-narrative-memory](skills/tracemem-weaving-narrative-memory/SKILL.md) | Build structured narrative memory systems from conversational traces using TraceMem's three-stage pipeline (segmentat... | 6/10 ★★★ | 229 |
| [amem4rec-leveraging-cross-user-similarity](skills/amem4rec-leveraging-cross-user-similarity/SKILL.md) | Build agentic recommendation systems that learn collaborative filtering signals through cross-user memory evolution -... | 5/10 ★★ | 273 |
| [attn-gs-attention-guided-context-compression](skills/attn-gs-attention-guided-context-compression/SKILL.md) | Compress long user contexts (profiles, histories, documents) into concise, high-quality summaries using attention-gui... | 5/10 ★★ | 158 |
| [clustering-driven-memory-compression-on-device](skills/clustering-driven-memory-compression-on-device/SKILL.md) | Compress user-specific memories for LLM personalization by clustering semantically similar memories and merging withi... | 5/10 ★★ | 254 |
| [cope-clipped-rope-as](skills/cope-clipped-rope-as/SKILL.md) | Implement CoPE (Clipped RoPE) soft clipping of low-frequency rotary positional embedding components to extend LLM con... | 5/10 ★★ | 215 |
| [dynamic-long-context-reasoning](skills/dynamic-long-context-reasoning/SKILL.md) | Process extremely long documents and contexts by compressing them into memory chunks, selectively retrieving relevant... | 5/10 ★★ | 210 |
| [how-personalized-memory-shape](skills/how-personalized-memory-shape/SKILL.md) | Rational preference utilization for personalized LLM assistants. Implements RP-Reasoner's pragmatic reasoning to sele... | 5/10 ★★ | 216 |
| [memcast-memory-driven-time-series](skills/memcast-memory-driven-time-series/SKILL.md) | Build memory-augmented time series forecasting systems using hierarchical experience storage (historical patterns, re... | 5/10 ★★ | 196 |
| [polarmem-training-free-polarized-latent](skills/polarmem-training-free-polarized-latent/SKILL.md) | Build polarized memory systems for multimodal agents that encode both positive and negative evidence as graph constra... | 5/10 ★★ | 183 |
| [read-as-human-compressing](skills/read-as-human-compressing/SKILL.md) | Compress long contexts using the RAM (Read As Human) strategy: partition text into segments, score relevance against ... | 5/10 ★★ | 242 |
| [shardmemo-masked-moe-routing](skills/shardmemo-masked-moe-routing/SKILL.md) | Implement ShardMemo-style tiered, sharded memory with masked Mixture-of-Experts routing for agentic LLM systems. Use ... | 5/10 ★★ | 176 |
| [locomo-plus-beyond-factual-cognitive-memory](skills/locomo-plus-beyond-factual-cognitive-memory/SKILL.md) | Build and evaluate cognitive memory systems for LLM dialogue agents that retain implicit user constraints (state, goa... | 4/10 ★★ | 216 |

---

## Knowledge Graphs

**18 skills** | Avg rating: 5.2/10

| Skill | Description | Rating | Lines |
|-------|-------------|--------|-------|
| [breaking-static-graph-context-aware](skills/breaking-static-graph-context-aware/SKILL.md) | Build query-adaptive knowledge graph retrieval systems using CatRAG's context-aware traversal. Transforms static KG-b... | 6/10 ★★★ | 170 |
| [generative-ontology-structured-knowledge](skills/generative-ontology-structured-knowledge/SKILL.md) | Constrain LLM generation with executable Pydantic schemas and multi-agent pipelines to produce structurally valid, do... | 6/10 ★★★ | 222 |
| [graph-based-agent-memory-taxonomy](skills/graph-based-agent-memory-taxonomy/SKILL.md) | Design and implement graph-based memory systems for LLM agents following the extraction-storage-retrieval-evolution l... | 6/10 ★★★ | 279 |
| [graphseek-next-generation-graph-analytics](skills/graphseek-next-generation-graph-analytics/SKILL.md) | Build LLM-powered graph analytics systems using the GraphSeek two-plane architecture: a Semantic Catalog for planning... | 6/10 ★★★ | 151 |
| [hugrag-hierarchical-causal-knowledge](skills/hugrag-hierarchical-causal-knowledge/SKILL.md) | Build hierarchical causal knowledge graphs for RAG pipelines that suppress spurious correlations and enable cross-doc... | 6/10 ★★★ | 168 |
| [vihermes-graph-grounded-multihop-question](skills/vihermes-graph-grounded-multihop-question/SKILL.md) | Build graph-grounded multihop QA systems over regulatory and hierarchically structured documents. Combines vector sim... | 6/10 ★★★ | 264 |
| [autonomous-business-system-neuro-symbolic](skills/autonomous-business-system-neuro-symbolic/SKILL.md) | Design and implement neuro-symbolic business automation systems that combine LLM agents with predicate-logic programm... | 5/10 ★★ | 249 |
| [context-augmented-code-generation-programming-know](skills/context-augmented-code-generation-programming-know/SKILL.md) | Enhance code generation with Programming Knowledge Graph (PKG) retrieval, tree pruning, and re-ranking. Uses fine-gra... | 5/10 ★★ | 191 |
| [core-comprehensive-ontological-relation](skills/core-comprehensive-ontological-relation/SKILL.md) | Detect and prevent semantic collapse in LLM outputs — where models fabricate spurious relationships between unrelated... | 5/10 ★★ | 216 |
| [domain-specific-knowledge-graphs-rag-enhanced](skills/domain-specific-knowledge-graphs-rag-enhanced/SKILL.md) | Build scope-matched knowledge graph RAG pipelines where retrieval precision beats breadth. Constructs domain-specific... | 5/10 ★★ | 217 |
| [graphagents-knowledge-graph-guided-agentic](skills/graphagents-knowledge-graph-guided-agentic/SKILL.md) | Build multi-agent pipelines that use knowledge graphs to guide LLM reasoning across domains. Agents specialize in pro... | 5/10 ★★ | 185 |
| [grounding-generative-planners-verifiable](skills/grounding-generative-planners-verifiable/SKILL.md) | Build neuro-symbolic safety verification pipelines using the VIRF (Verifiable Iterative Refinement Framework) pattern... | 5/10 ★★ | 176 |
| [kg-craft-knowledge-graph-based-contrastive](skills/kg-craft-knowledge-graph-based-contrastive/SKILL.md) | Fact-check claims using knowledge graph-based contrastive reasoning. Constructs a KG from claims and evidence sources... | 5/10 ★★ | 197 |
| [koral-knowledge-graph-guided](skills/koral-knowledge-graph-guided/SKILL.md) | Build Knowledge Graph-guided LLM reasoning pipelines for operational telemetry analysis. Combines a Literature KG (ex... | 5/10 ★★ | 247 |
| [medspeak-knowledge-graph-aided-asr](skills/medspeak-knowledge-graph-aided-asr/SKILL.md) | Build knowledge-graph-aided ASR error correction pipelines for medical speech, using phonetic similarity + semantic r... | 5/10 ★★ | 262 |
| [ontology-to-tools-compilation-executable-semantic-](skills/ontology-to-tools-compilation-executable-semantic/SKILL.md) | Compile domain ontologies (OWL/RDFS/JSON-LD schemas) into executable tool interfaces with embedded semantic constrain... | 5/10 ★★ | 192 |
| [pruning-minimal-reasoning-graphs](skills/pruning-minimal-reasoning-graphs/SKILL.md) | Build and maintain compact reasoning graphs for retrieval-augmented generation (RAG) that persist across queries, pru... | 5/10 ★★ | 154 |
| [toward-culturally-aligned-ontology-guided](skills/toward-culturally-aligned-ontology-guided/SKILL.md) | Ontology-guided multi-agent reasoning for culturally aligned LLM outputs. Use when building systems that must respect... | 3/10 ★★ | 190 |

---

## Explainability

**7 skills** | Avg rating: 5.0/10

| Skill | Description | Rating | Lines |
|-------|-------------|--------|-------|
| [agenticsimlaw-juvenile-courtroom-multi-agent](skills/agenticsimlaw-juvenile-courtroom-multi-agent/SKILL.md) | Structured multi-agent courtroom debate for explainable high-stakes tabular decisions. Use when: 'set up a multi-agen... | 6/10 ★★★ | 181 |
| [interpreting-agentic-systems-beyond](skills/interpreting-agentic-systems-beyond/SKILL.md) | Audit and instrument agentic AI systems for system-level interpretability and accountability. Embeds traceability, ca... | 6/10 ★★★ | 329 |
| [xlist-hate-checklist-based-framework-interpretable](skills/xlist-hate-checklist-based-framework-interpretable/SKILL.md) | Decompose hate speech detection into a checklist of ten concept-level binary questions answered independently by an L... | 6/10 ★★★ | 229 |
| [addressing-explainability-generative-ai](skills/addressing-explainability-generative-ai/SKILL.md) | Explain generative AI outputs using the gSMILE perturbation-based attribution framework. Builds local surrogate model... | 5/10 ★★ | 222 |
| [from-features-actions-explainability](skills/from-features-actions-explainability/SKILL.md) | Diagnose and explain failures in agentic AI systems using trace-based rubric evaluation, bridging static feature attr... | 5/10 ★★ | 207 |
| [xai-clip-roi-guided-perturbation-framework](skills/xai-clip-roi-guided-perturbation-framework/SKILL.md) | Build ROI-guided perturbation pipelines for explainable medical image segmentation using CLIP embeddings. Generates b... | 4/10 ★★ | 226 |
| [agentxray-white-boxing-agentic-systems](skills/agentxray-white-boxing-agentic-systems/SKILL.md) | Reverse-engineer black-box agentic systems into editable, interpretable workflows using search-based reconstruction. ... | 3/10 ★★ | 168 |

---

## Fine-tuning & Training

**7 skills** | Avg rating: 4.3/10

| Skill | Description | Rating | Lines |
|-------|-------------|--------|-------|
| [datachef-cooking-up-optimal](skills/datachef-cooking-up-optimal/SKILL.md) | Automate data recipe generation for LLM fine-tuning and adaptation. Generates executable data processing pipelines (f... | 5/10 ★★ | 207 |
| [layer-wise-lora-fine-tuning-similarity](skills/layer-wise-lora-fine-tuning-similarity/SKILL.md) | Selectively apply LoRA adapters to only the most important transformer layers using CKA similarity-based layer import... | 5/10 ★★ | 221 |
| [automated-customization-enterprise-code](skills/automated-customization-enterprise-code/SKILL.md) | Customize LLMs for enterprise code repositories using semantic scopes -- automatically partition codebases into meani... | 4/10 ★★ | 153 |
| [better-as-generators-than](skills/better-as-generators-than/SKILL.md) | Generate synthetic labeled datasets with LLMs to train smaller, cheaper classifiers -- especially for low-resource la... | 4/10 ★★ | 178 |
| [explicit-multi-head-attention-inter-head](skills/explicit-multi-head-attention-inter-head/SKILL.md) | Implement Multi-head Explicit Attention (MEA) with inter-head interaction for Transformer models. Adds Head-level Lin... | 4/10 ★★ | 217 |
| [learning-rate-matters-vanilla](skills/learning-rate-matters-vanilla/SKILL.md) | Configure optimal learning rates for LoRA fine-tuning of LLMs. Generates hyperparameter search configs, training scri... | 4/10 ★★ | 203 |
| [privacy-collapse-benign-fine-tuning](skills/privacy-collapse-benign-fine-tuning/SKILL.md) | Audit fine-tuning datasets and pipelines for privacy collapse — the silent failure where benign training data degrade... | 4/10 ★★ | 195 |

---

## All Skills (Alphabetical)

Complete list of all 651 curated skills.

| Skill | Category | Rating | Lines |
|-------|----------|--------|-------|
| [3-secbench-large-scale-evaluation-suite-security](skills/3-secbench-large-scale-evaluation-suite-security/SKILL.md) | Evaluation & Benchmarking | 4/10 | 182 |
| [a-mapreduce-executing-wide-search](skills/a-mapreduce-executing-wide-search/SKILL.md) | Agentic Systems | 7/10 | 196 |
| [a-rag-scaling-agentic-retrieval-augmented](skills/a-rag-scaling-agentic-retrieval-augmented/SKILL.md) | RAG & Retrieval | 7/10 | 253 |
| [a2rag-adaptive-agentic-graph](skills/a2rag-adaptive-agentic-graph/SKILL.md) | RAG & Retrieval | 6/10 | 229 |
| [aacr-bench-evaluating-automatic-code](skills/aacr-bench-evaluating-automatic-code/SKILL.md) | Code & Software Engineering | 7/10 | 187 |
| [accelerating-social-science-research](skills/accelerating-social-science-research/SKILL.md) | Data Processing | 6/10 | 177 |
| [acegrpo-adaptive-curriculum-group](skills/acegrpo-adaptive-curriculum-group/SKILL.md) | Agentic Systems | 5/10 | 220 |
| [adaptbpe-general-purpose-specialized](skills/adaptbpe-general-purpose-specialized/SKILL.md) | NLP & Text | 5/10 | 202 |
| [adaptive-confidence-gating-multi-agent](skills/adaptive-confidence-gating-multi-agent/SKILL.md) | Reasoning & Chain-of-Thought | 6/10 | 184 |
| [adareasoner-dynamic-tool-orchestration](skills/adareasoner-dynamic-tool-orchestration/SKILL.md) | Agentic Systems | 6/10 | 168 |
| [addressing-explainability-generative-ai](skills/addressing-explainability-generative-ai/SKILL.md) | Explainability | 5/10 | 222 |
| [adoption-use-at-academic](skills/adoption-use-at-academic/SKILL.md) | Domain-Specific | 5/10 | 389 |
| [aegis-governance-integrity-security](skills/aegis-governance-integrity-security/SKILL.md) | Security & Safety | 6/10 | 237 |
| [aero-autonomous-evolutionary-reasoning](skills/aero-autonomous-evolutionary-reasoning/SKILL.md) | Reasoning & Chain-of-Thought | 5/10 | 160 |
| [agent-based-software-artifact-evaluation](skills/agent-based-software-artifact-evaluation/SKILL.md) | Evaluation & Benchmarking | 6/10 | 203 |
| [agent-fence-mapping-security-vulnerabilities](skills/agent-fence-mapping-security-vulnerabilities/SKILL.md) | Security & Safety | 7/10 | 211 |
| [agent-primitives-reusable-latent](skills/agent-primitives-reusable-latent/SKILL.md) | Multi-Agent Systems | 6/10 | 252 |
| [agent2agent-threats-safety-critical-assistants](skills/agent2agent-threats-safety-critical-assistants/SKILL.md) | Security & Safety | 6/10 | 207 |
| [agentcgroup-understanding-controlling-os](skills/agentcgroup-understanding-controlling-os/SKILL.md) | Efficiency & Optimization | 6/10 | 231 |
| [agentcpm-report-interleaving-drafting-deepening](skills/agentcpm-report-interleaving-drafting-deepening/SKILL.md) | NLP & Text | 6/10 | 232 |
| [agentdog-diagnostic-guardrail-framework](skills/agentdog-diagnostic-guardrail-framework/SKILL.md) | Security & Safety | 5/10 | 340 |
| [agentdrive-open-benchmark-dataset](skills/agentdrive-open-benchmark-dataset/SKILL.md) | Evaluation & Benchmarking | 5/10 | 284 |
| [agentic-ai-healthcare-medicine](skills/agentic-ai-healthcare-medicine/SKILL.md) | Domain-Specific | 4/10 | 274 |
| [agentic-very-long-video](skills/agentic-very-long-video/SKILL.md) | Agentic Systems | 5/10 | 240 |
| [agenticpay-multi-agent-negotiation-system](skills/agenticpay-multi-agent-negotiation-system/SKILL.md) | Multi-Agent Systems | 5/10 | 226 |
| [agenticscr-an-autonomous-agentic](skills/agenticscr-an-autonomous-agentic/SKILL.md) | Security & Safety | 8/10 | 177 |
| [agenticsimlaw-juvenile-courtroom-multi-agent](skills/agenticsimlaw-juvenile-courtroom-multi-agent/SKILL.md) | Explainability | 6/10 | 181 |
| [agentsm-semantic-memory-agentic](skills/agentsm-semantic-memory-agentic/SKILL.md) | Memory & Context | 7/10 | 213 |
| [agentstepper-interactive-debugging-software](skills/agentstepper-interactive-debugging-software/SKILL.md) | Agentic Systems | 7/10 | 197 |
| [agentsys-secure-dynamic-agents](skills/agentsys-secure-dynamic-agents/SKILL.md) | Security & Safety | 7/10 | 191 |
| [agenttrace-structured-logging-framework](skills/agenttrace-structured-logging-framework/SKILL.md) | Agentic Systems | 7/10 | 152 |
| [agentxray-white-boxing-agentic-systems](skills/agentxray-white-boxing-agentic-systems/SKILL.md) | Explainability | 3/10 | 168 |
| [agyn-multi-agent-system-team-based](skills/agyn-multi-agent-system-team-based/SKILL.md) | Multi-Agent Systems | 7/10 | 202 |
| [ai-agent-for-reverseengineering](skills/ai-agent-for-reverseengineering/SKILL.md) | Domain-Specific | 7/10 | 205 |
| [ai-agent-systems-supply](skills/ai-agent-systems-supply/SKILL.md) | Domain-Specific | 5/10 | 277 |
| [ai-my-values-user](skills/ai-my-values-user/SKILL.md) | Agentic Systems | 5/10 | 236 |
| [aiano-enhancing-information-retrieval](skills/aiano-enhancing-information-retrieval/SKILL.md) | Data Processing | 6/10 | 161 |
| [aidev-studying-ai-coding](skills/aidev-studying-ai-coding/SKILL.md) | Evaluation & Benchmarking | 6/10 | 195 |
| [alertguardian-intelligent-alert-life-cycle](skills/alertguardian-intelligent-alert-life-cycle/SKILL.md) | Domain-Specific | 5/10 | 201 |
| [alienlm-alienization-api-boundary-privacy](skills/alienlm-alienization-api-boundary-privacy/SKILL.md) | Security & Safety | 4/10 | 202 |
| [alignagent-adaptive-learner-intelligence](skills/alignagent-adaptive-learner-intelligence/SKILL.md) | Domain-Specific | 5/10 | 319 |
| [aligncoder-aligning-retrieval-target](skills/aligncoder-aligning-retrieval-target/SKILL.md) | RAG & Retrieval | 7/10 | 206 |
| [alrm-agentic-robotic-manipulation](skills/alrm-agentic-robotic-manipulation/SKILL.md) | Agentic Systems | 5/10 | 131 |
| [ama-adaptive-memory-multi-agent](skills/ama-adaptive-memory-multi-agent/SKILL.md) | Memory & Context | 6/10 | 224 |
| [amem4rec-leveraging-cross-user-similarity](skills/amem4rec-leveraging-cross-user-similarity/SKILL.md) | Memory & Context | 5/10 | 273 |
| [an-cost-efficient-agentic-framework](skills/an-cost-efficient-agentic-framework/SKILL.md) | Security & Safety | 7/10 | 202 |
| [analyticsgpt-workflow-scientometric-question](skills/analyticsgpt-workflow-scientometric-question/SKILL.md) | Domain-Specific | 5/10 | 306 |
| [anonymization-enhanced-privacy-protection-mobile-g](skills/anonymization-enhanced-privacy-protection-mobile-g/SKILL.md) | Security & Safety | 5/10 | 178 |
| [aorchestra-automating-sub-agent-creation](skills/aorchestra-automating-sub-agent-creation/SKILL.md) | Multi-Agent Systems | 7/10 | 203 |
| [apex-agents](skills/apex-agents/SKILL.md) | Evaluation & Benchmarking | 4/10 | 223 |
| [are-open-weight-ready-social](skills/are-open-weight-ready-social/SKILL.md) | NLP & Text | 5/10 | 264 |
| [arkeval-benchmarking-evaluating-automated](skills/arkeval-benchmarking-evaluating-automated/SKILL.md) | Domain-Specific | 4/10 | 196 |
| [artificial-intelligence-open-source](skills/artificial-intelligence-open-source/SKILL.md) | Code & Software Engineering | 4/10 | 210 |
| [assessing-business-process-modeling](skills/assessing-business-process-modeling/SKILL.md) | Domain-Specific | 5/10 | 229 |
| [assessing-quality-mental-health](skills/assessing-quality-mental-health/SKILL.md) | Evaluation & Benchmarking | 4/10 | 374 |
| [assessment-generative-named-entity](skills/assessment-generative-named-entity/SKILL.md) | NLP & Text | 5/10 | 245 |
| [atomic-information-flow-network](skills/atomic-information-flow-network/SKILL.md) | RAG & Retrieval | 5/10 | 197 |
| [attn-gs-attention-guided-context-compression](skills/attn-gs-attention-guided-context-compression/SKILL.md) | Memory & Context | 5/10 | 158 |
| [automated-customization-enterprise-code](skills/automated-customization-enterprise-code/SKILL.md) | Fine-tuning & Training | 4/10 | 153 |
| [automated-multiple-mini-interview](skills/automated-multiple-mini-interview/SKILL.md) | Evaluation & Benchmarking | 6/10 | 189 |
| [automated-rubrics-reliable-evaluation](skills/automated-rubrics-reliable-evaluation/SKILL.md) | Evaluation & Benchmarking | 5/10 | 180 |
| [automated-structural-testing-llm-based](skills/automated-structural-testing-llm-based/SKILL.md) | Evaluation & Benchmarking | 8/10 | 242 |
| [automating-computational-reproducibility-social](skills/automating-computational-reproducibility-social/SKILL.md) | Domain-Specific | 6/10 | 189 |
| [autonomous-business-system-neuro-symbolic](skills/autonomous-business-system-neuro-symbolic/SKILL.md) | Knowledge Graphs | 5/10 | 249 |
| [autonomous-data-processing-meta-agents](skills/autonomous-data-processing-meta-agents/SKILL.md) | Data Processing | 7/10 | 216 |
| [autonomous-multi-agent-ai-high-throughput](skills/autonomous-multi-agent-ai-high-throughput/SKILL.md) | Multi-Agent Systems | 4/10 | 189 |
| [autoregressive-yet-revisable-decoding-revision](skills/autoregressive-yet-revisable-decoding-revision/SKILL.md) | Security & Safety | 5/10 | 201 |
| [avenir-web-human-experience-imitating-multimodal-w](skills/avenir-web-human-experience-imitating-multimodal-w/SKILL.md) | Agentic Systems | 6/10 | 375 |
| [bass-benchmarking-audio-lms](skills/bass-benchmarking-audio-lms/SKILL.md) | Evaluation & Benchmarking | 4/10 | 260 |
| [batcoder-self-supervised-bidirectional-code-docume](skills/batcoder-self-supervised-bidirectional-code-docume/SKILL.md) | Code & Software Engineering | 6/10 | 207 |
| [bayesflow-probability-inference-framework](skills/bayesflow-probability-inference-framework/SKILL.md) | Agentic Systems | 5/10 | 204 |
| [benchmarking-abap-code-generation](skills/benchmarking-abap-code-generation/SKILL.md) | Domain-Specific | 6/10 | 183 |
| [benchmarking-pairwise-causal-discovery](skills/benchmarking-pairwise-causal-discovery/SKILL.md) | NLP & Text | 5/10 | 204 |
| [benchmarking-reward-hack-detection](skills/benchmarking-reward-hack-detection/SKILL.md) | Evaluation & Benchmarking | 5/10 | 190 |
| [benchmarking-text-to-python-against-text-to-sql](skills/benchmarking-text-to-python-against-text-to-sql/SKILL.md) | Data Processing | 7/10 | 188 |
| [benchmarking-uncertainty-calibration-long-form](skills/benchmarking-uncertainty-calibration-long-form/SKILL.md) | Evaluation & Benchmarking | 5/10 | 210 |
| [benchmarking-zero-shot-few-shot-phishing](skills/benchmarking-zero-shot-few-shot-phishing/SKILL.md) | Security & Safety | 5/10 | 215 |
| [better-as-generators-than](skills/better-as-generators-than/SKILL.md) | Fine-tuning & Training | 4/10 | 178 |
| [beyond-accuracy-cognitive-load](skills/beyond-accuracy-cognitive-load/SKILL.md) | Agentic Systems | 6/10 | 218 |
| [beyond-blame-rethinking-szz](skills/beyond-blame-rethinking-szz/SKILL.md) | Code & Software Engineering | 7/10 | 154 |
| [beyond-function-level-analysis-context-aware](skills/beyond-function-level-analysis-context-aware/SKILL.md) | Security & Safety | 8/10 | 170 |
| [beyond-holistic-scores-automatic](skills/beyond-holistic-scores-automatic/SKILL.md) | NLP & Text | 5/10 | 201 |
| [beyond-instrumental-substitutive-paradigms](skills/beyond-instrumental-substitutive-paradigms/SKILL.md) | Evaluation & Benchmarking | 3/10 | 189 |
| [beyond-needles-illusion-decoupled](skills/beyond-needles-illusion-decoupled/SKILL.md) | Evaluation & Benchmarking | 5/10 | 191 |
| [beyond-translation-cross-cultural-meme](skills/beyond-translation-cross-cultural-meme/SKILL.md) | Multimodal | 4/10 | 170 |
| [biases-blind-spot-detecting](skills/biases-blind-spot-detecting/SKILL.md) | Evaluation & Benchmarking | 6/10 | 194 |
| [biasscope-automated-detection-bias](skills/biasscope-automated-detection-bias/SKILL.md) | Evaluation & Benchmarking | 5/10 | 206 |
| [bioace-automated-framework-biomedical](skills/bioace-automated-framework-biomedical/SKILL.md) | Evaluation & Benchmarking | 4/10 | 238 |
| [birdturk-adaptation-bird-text-to-sql](skills/birdturk-adaptation-bird-text-to-sql/SKILL.md) | NLP & Text | 4/10 | 242 |
| [blind-gods-broken-screens](skills/blind-gods-broken-screens/SKILL.md) | Multi-Agent Systems | 6/10 | 219 |
| [breaking-protocol-security-analysis](skills/breaking-protocol-security-analysis/SKILL.md) | Security & Safety | 7/10 | 264 |
| [breaking-static-graph-context-aware](skills/breaking-static-graph-context-aware/SKILL.md) | Knowledge Graphs | 6/10 | 170 |
| [bridging-arithmetic-gap-cognitive](skills/bridging-arithmetic-gap-cognitive/SKILL.md) | Reasoning & Chain-of-Thought | 7/10 | 222 |
| [bridging-modality-gap-roadside](skills/bridging-modality-gap-roadside/SKILL.md) | Domain-Specific | 5/10 | 211 |
| [bridging-online-offline-rl](skills/bridging-online-offline-rl/SKILL.md) | Reasoning & Chain-of-Thought | 4/10 | 160 |
| [c-mop-integrating-momentum-boundary-aware](skills/c-mop-integrating-momentum-boundary-aware/SKILL.md) | Prompt Engineering | 7/10 | 179 |
| [c2rope-causal-continuous-rotary-positional](skills/c2rope-causal-continuous-rotary-positional/SKILL.md) | Domain-Specific | 5/10 | 235 |
| [calliope-tts-based-narrated-e-book](skills/calliope-tts-based-narrated-e-book/SKILL.md) | Multimodal | 5/10 | 178 |
| [cam-causality-based-analysis-framework](skills/cam-causality-based-analysis-framework/SKILL.md) | Multi-Agent Systems | 4/10 | 153 |
| [can-clean-up-mess](skills/can-clean-up-mess/SKILL.md) | Data Processing | 8/10 | 164 |
| [can-implement-agent-based-odd-based](skills/can-implement-agent-based-odd-based/SKILL.md) | Domain-Specific | 6/10 | 208 |
| [can-post-training-transform-causal](skills/can-post-training-transform-causal/SKILL.md) | Reasoning & Chain-of-Thought | 5/10 | 179 |
| [can-reasoning-be-trusted](skills/can-reasoning-be-trusted/SKILL.md) | Evaluation & Benchmarking | 5/10 | 203 |
| [can-small-handle-context-summarized](skills/can-small-handle-context-summarized/SKILL.md) | NLP & Text | 4/10 | 254 |
| [can-we-classify-flaky](skills/can-we-classify-flaky/SKILL.md) | Code & Software Engineering | 7/10 | 182 |
| [canonical-intermediate-representation-llm-based](skills/canonical-intermediate-representation-llm-based/SKILL.md) | Domain-Specific | 7/10 | 237 |
| [capture-flags-family-based-evaluation](skills/capture-flags-family-based-evaluation/SKILL.md) | Evaluation & Benchmarking | 5/10 | 165 |
| [causalt5k-diagnosing-informing-refusal](skills/causalt5k-diagnosing-informing-refusal/SKILL.md) | Reasoning & Chain-of-Thought | 5/10 | 161 |
| [chain-mindset-reasoning-adaptive](skills/chain-mindset-reasoning-adaptive/SKILL.md) | Reasoning & Chain-of-Thought | 5/10 | 177 |
| [chain-simulation-dual-mode-reasoning](skills/chain-simulation-dual-mode-reasoning/SKILL.md) | Reasoning & Chain-of-Thought | 6/10 | 175 |
| [chart-specification-structural-representations](skills/chart-specification-structural-representations/SKILL.md) | Multimodal | 7/10 | 344 |
| [chatting-images-introspective-visual](skills/chatting-images-introspective-visual/SKILL.md) | Multimodal | 6/10 | 186 |
| [chipbench-next-step-benchmark-evaluating](skills/chipbench-next-step-benchmark-evaluating/SKILL.md) | Domain-Specific | 5/10 | 225 |
| [chunking-retrieval-re-ranking-empirical-evaluation](skills/chunking-retrieval-re-ranking-empirical-evaluation/SKILL.md) | RAG & Retrieval | 7/10 | 230 |
| [ci4a-semantic-component-interfaces](skills/ci4a-semantic-component-interfaces/SKILL.md) | Agentic Systems | 7/10 | 272 |
| [closing-reasoning-gaps-clinical](skills/closing-reasoning-gaps-clinical/SKILL.md) | Reasoning & Chain-of-Thought | 5/10 | 196 |
| [clustering-driven-memory-compression-on-device](skills/clustering-driven-memory-compression-on-device/SKILL.md) | Memory & Context | 5/10 | 254 |
| [co-redteam-orchestrated-security-discovery](skills/co-redteam-orchestrated-security-discovery/SKILL.md) | Security & Safety | 7/10 | 197 |
| [code2world-gui-world-renderable](skills/code2world-gui-world-renderable/SKILL.md) | Multimodal | 6/10 | 292 |
| [codecircuit-inferring-llm-generated-code](skills/codecircuit-inferring-llm-generated-code/SKILL.md) | Code & Software Engineering | 6/10 | 195 |
| [codeocr-effectiveness-vision-code](skills/codeocr-effectiveness-vision-code/SKILL.md) | Efficiency & Optimization | 5/10 | 241 |
| [cognitive-platform-engineering-autonomous](skills/cognitive-platform-engineering-autonomous/SKILL.md) | Domain-Specific | 5/10 | 247 |
| [cognitively-diverse-multiple-choice-question](skills/cognitively-diverse-multiple-choice-question/SKILL.md) | NLP & Text | 5/10 | 177 |
| [commcp-multi-agent-coordination-llm-based](skills/commcp-multi-agent-coordination-llm-based/SKILL.md) | Multi-Agent Systems | 5/10 | 225 |
| [compact-hypercube-embeddings-fast](skills/compact-hypercube-embeddings-fast/SKILL.md) | RAG & Retrieval | 5/10 | 192 |
| [compactrag-reducing-calls-token](skills/compactrag-reducing-calls-token/SKILL.md) | RAG & Retrieval | 5/10 | 194 |
| [compar-ia-french-governments](skills/compar-ia-french-governments/SKILL.md) | Evaluation & Benchmarking | 5/10 | 310 |
| [comparing-ai-coding-agents](skills/comparing-ai-coding-agents/SKILL.md) | Evaluation & Benchmarking | 5/10 | 207 |
| [compass-contrastive-learning-automated](skills/compass-contrastive-learning-automated/SKILL.md) | Evaluation & Benchmarking | 5/10 | 180 |
| [completing-missing-annotation-multi-agent](skills/completing-missing-annotation-multi-agent/SKILL.md) | Evaluation & Benchmarking | 5/10 | 251 |
| [comprehensive-comparison-rag-methods](skills/comprehensive-comparison-rag-methods/SKILL.md) | RAG & Retrieval | 7/10 | 167 |
| [comprehensive-evaluation-software-engineering](skills/comprehensive-evaluation-software-engineering/SKILL.md) | Evaluation & Benchmarking | 5/10 | 285 |
| [computational-approach-visual-metonymy](skills/computational-approach-visual-metonymy/SKILL.md) | Multimodal | 4/10 | 179 |
| [conceptual-cultural-index-metric](skills/conceptual-cultural-index-metric/SKILL.md) | NLP & Text | 4/10 | 246 |
| [consistency-meets-verification-enhancing](skills/consistency-meets-verification-enhancing/SKILL.md) | Code & Software Engineering | 7/10 | 178 |
| [constitutional-spec-driven-development-enforcing](skills/constitutional-spec-driven-development-enforcing/SKILL.md) | Security & Safety | 7/10 | 258 |
| [constrained-process-maps-multi-agent](skills/constrained-process-maps-multi-agent/SKILL.md) | Multi-Agent Systems | 6/10 | 244 |
| [constructing-multi-label-hierarchical-classificati](skills/constructing-multi-label-hierarchical-classificati/SKILL.md) | Security & Safety | 5/10 | 226 |
| [context-augmented-code-generation-programming-know](skills/context-augmented-code-generation-programming-know/SKILL.md) | Knowledge Graphs | 5/10 | 191 |
| [context-sensitive-pointer-analysis-arkts](skills/context-sensitive-pointer-analysis-arkts/SKILL.md) | Security & Safety | 5/10 | 223 |
| [contextevolve-multi-agent-context-compression](skills/contextevolve-multi-agent-context-compression/SKILL.md) | Efficiency & Optimization | 5/10 | 175 |
| [contextual-drag-errors-context](skills/contextual-drag-errors-context/SKILL.md) | Reasoning & Chain-of-Thought | 6/10 | 249 |
| [controlling-output-rankings-generative](skills/controlling-output-rankings-generative/SKILL.md) | Prompt Engineering | 5/10 | 245 |
| [conversation-non-verifiable-learning-self-evolving](skills/conversation-non-verifiable-learning-self-evolving/SKILL.md) | Reasoning & Chain-of-Thought | 6/10 | 241 |
| [convexbench-recognize-convex-functions](skills/convexbench-recognize-convex-functions/SKILL.md) | Reasoning & Chain-of-Thought | 6/10 | 161 |
| [cope-clipped-rope-as](skills/cope-clipped-rope-as/SKILL.md) | Memory & Context | 5/10 | 215 |
| [core-comprehensive-ontological-relation](skills/core-comprehensive-ontological-relation/SKILL.md) | Knowledge Graphs | 5/10 | 216 |
| [corefine-confidence-guided-self-refinement-adaptiv](skills/corefine-confidence-guided-self-refinement-adaptiv/SKILL.md) | Reasoning & Chain-of-Thought | 6/10 | 176 |
| [corpusqa-10-million-token](skills/corpusqa-10-million-token/SKILL.md) | Data Processing | 7/10 | 179 |
| [cost-aware-selection-text-classification](skills/cost-aware-selection-text-classification/SKILL.md) | NLP & Text | 5/10 | 174 |
| [cost-efficient-rag-entity-matching](skills/cost-efficient-rag-entity-matching/SKILL.md) | RAG & Retrieval | 6/10 | 187 |
| [covagent-overcoming-30-curse](skills/covagent-overcoming-30-curse/SKILL.md) | Security & Safety | 6/10 | 174 |
| [cowork-x-experience-optimized-co-evolution-multi-a](skills/cowork-x-experience-optimized-co-evolution-multi-a/SKILL.md) | Multi-Agent Systems | 4/10 | 150 |
| [craft-calibrated-reasoning-answer-faithful](skills/craft-calibrated-reasoning-answer-faithful/SKILL.md) | RAG & Retrieval | 6/10 | 192 |
| [creditaudit-2textnd-dimension-evaluation](skills/creditaudit-2textnd-dimension-evaluation/SKILL.md) | Evaluation & Benchmarking | 4/10 | 211 |
| [cross-lingual-stability-judges-under](skills/cross-lingual-stability-judges-under/SKILL.md) | Evaluation & Benchmarking | 5/10 | 199 |
| [ctrlcot-dual-granularity-chain-of-thought-compress](skills/ctrlcot-dual-granularity-chain-of-thought-compress/SKILL.md) | Reasoning & Chain-of-Thought | 5/10 | 157 |
| [cua-skill-develop-skills-computer](skills/cua-skill-develop-skills-computer/SKILL.md) | Agentic Systems | 6/10 | 297 |
| [culturally-grounded-personas-characterization](skills/culturally-grounded-personas-characterization/SKILL.md) | Prompt Engineering | 4/10 | 202 |
| [curiosity-driven-knowledge-retrieval](skills/curiosity-driven-knowledge-retrieval/SKILL.md) | Agentic Systems | 5/10 | 219 |
| [cutting-gordian-knot-detecting](skills/cutting-gordian-knot-detecting/SKILL.md) | Security & Safety | 7/10 | 212 |
| [cve-factory-scaling-expert-level-agentic](skills/cve-factory-scaling-expert-level-agentic/SKILL.md) | Security & Safety | 6/10 | 240 |
| [cvedrl-code-verifier-difficulty-aware](skills/cvedrl-code-verifier-difficulty-aware/SKILL.md) | Evaluation & Benchmarking | 6/10 | 195 |
| [dancing-chains-strategic-persuasion](skills/dancing-chains-strategic-persuasion/SKILL.md) | Prompt Engineering | 6/10 | 266 |
| [darl-encouraging-diverse-answers](skills/darl-encouraging-diverse-answers/SKILL.md) | Prompt Engineering | 5/10 | 286 |
| [dart-diffusion-inspired-speculative-decoding](skills/dart-diffusion-inspired-speculative-decoding/SKILL.md) | Efficiency & Optimization | 4/10 | 241 |
| [darwin-dynamic-agentically-rewriting](skills/darwin-dynamic-agentically-rewriting/SKILL.md) | Agentic Systems | 4/10 | 167 |
| [datachef-cooking-up-optimal](skills/datachef-cooking-up-optimal/SKILL.md) | Fine-tuning & Training | 5/10 | 207 |
| [datacross-unified-benchmark-agent](skills/datacross-unified-benchmark-agent/SKILL.md) | Data Processing | 6/10 | 203 |
| [david-vs-goliath-verifiable](skills/david-vs-goliath-verifiable/SKILL.md) | Security & Safety | 5/10 | 165 |
| [davinci-agency-unlocking-long-horizon-agency](skills/davinci-agency-unlocking-long-horizon-agency/SKILL.md) | Code & Software Engineering | 8/10 | 181 |
| [davinci-dev-agent-native-mid-training-software](skills/davinci-dev-agent-native-mid-training-software/SKILL.md) | Code & Software Engineering | 7/10 | 171 |
| [debugging-code-world](skills/debugging-code-world/SKILL.md) | Reasoning & Chain-of-Thought | 7/10 | 171 |
| [decomposing-reasoning-efficiency](skills/decomposing-reasoning-efficiency/SKILL.md) | Evaluation & Benchmarking | 4/10 | 248 |
| [decoupling-skeleton-flesh-multimodal](skills/decoupling-skeleton-flesh-multimodal/SKILL.md) | Multimodal | 6/10 | 180 |
| [deep-researcher-sequential-plan](skills/deep-researcher-sequential-plan/SKILL.md) | Reasoning & Chain-of-Thought | 7/10 | 194 |
| [deep-search-hierarchical-meta-cognitive](skills/deep-search-hierarchical-meta-cognitive/SKILL.md) | Agentic Systems | 5/10 | 188 |
| [deepera-deep-evidence-reranking](skills/deepera-deep-evidence-reranking/SKILL.md) | RAG & Retrieval | 7/10 | 223 |
| [deepimagesearch-benchmarking-multimodal-agents](skills/deepimagesearch-benchmarking-multimodal-agents/SKILL.md) | RAG & Retrieval | 4/10 | 198 |
| [deepplanning-benchmarking-long-horizon-agentic](skills/deepplanning-benchmarking-long-horizon-agentic/SKILL.md) | Agentic Systems | 5/10 | 155 |
| [deepread-document-structure-aware-reasoning](skills/deepread-document-structure-aware-reasoning/SKILL.md) | RAG & Retrieval | 6/10 | 187 |
| [deltaevolve-accelerating-scientific-discovery](skills/deltaevolve-accelerating-scientific-discovery/SKILL.md) | Efficiency & Optimization | 6/10 | 187 |
| [dep-search-learning-dependency-aware-reasoning](skills/dep-search-learning-dependency-aware-reasoning/SKILL.md) | Reasoning & Chain-of-Thought | 6/10 | 213 |
| [detecting-correcting-hallucinations-llm-generated](skills/detecting-correcting-hallucinations-llm-generated/SKILL.md) | Code & Software Engineering | 7/10 | 260 |
| [devops-gym-benchmarking-ai-agents](skills/devops-gym-benchmarking-ai-agents/SKILL.md) | Code & Software Engineering | 6/10 | 170 |
| [dial-summer-structured-evaluation-framework](skills/dial-summer-structured-evaluation-framework/SKILL.md) | Evaluation & Benchmarking | 5/10 | 244 |
| [diffusion-pretrained-dense-contextual-embeddings](skills/diffusion-pretrained-dense-contextual-embeddings/SKILL.md) | RAG & Retrieval | 5/10 | 183 |
| [discovering-high-level-patterns](skills/discovering-high-level-patterns/SKILL.md) | Data Processing | 4/10 | 190 |
| [discovering-process-outcome-credit-multi-step](skills/discovering-process-outcome-credit-multi-step/SKILL.md) | Reasoning & Chain-of-Thought | 5/10 | 190 |
| [discoverllm-executing-intents-discovering](skills/discoverllm-executing-intents-discovering/SKILL.md) | Prompt Engineering | 7/10 | 227 |
| [diverge-diversity-enhanced-rag-open-ended](skills/diverge-diversity-enhanced-rag-open-ended/SKILL.md) | RAG & Retrieval | 6/10 | 177 |
| [dllm-agent-see-farther](skills/dllm-agent-see-farther/SKILL.md) | Agentic Systems | 6/10 | 163 |
| [do-reasoning-ask-questions](skills/do-reasoning-ask-questions/SKILL.md) | Prompt Engineering | 6/10 | 183 |
| [do-truly-benefit-longer](skills/do-truly-benefit-longer/SKILL.md) | Efficiency & Optimization | 5/10 | 252 |
| [do-vlms-have-moral](skills/do-vlms-have-moral/SKILL.md) | Evaluation & Benchmarking | 4/10 | 211 |
| [doc2spec-synthesizing-formal-programming](skills/doc2spec-synthesizing-formal-programming/SKILL.md) | Code & Software Engineering | 6/10 | 177 |
| [docksmith-scaling-reliable-coding](skills/docksmith-scaling-reliable-coding/SKILL.md) | Code & Software Engineering | 8/10 | 201 |
| [domain-specific-knowledge-graphs-rag-enhanced](skills/domain-specific-knowledge-graphs-rag-enhanced/SKILL.md) | Knowledge Graphs | 5/10 | 217 |
| [dr-kernel-reinforcement-learning-done](skills/dr-kernel-reinforcement-learning-done/SKILL.md) | Efficiency & Optimization | 7/10 | 200 |
| [draincode-stealthy-energy-consumption](skills/draincode-stealthy-energy-consumption/SKILL.md) | Security & Safety | 5/10 | 221 |
| [drpg-decompose-retrieve-plan](skills/drpg-decompose-retrieve-plan/SKILL.md) | NLP & Text | 6/10 | 253 |
| [dynamic-framework-collaborative-learning](skills/dynamic-framework-collaborative-learning/SKILL.md) | Domain-Specific | 5/10 | 214 |
| [dynamic-long-context-reasoning](skills/dynamic-long-context-reasoning/SKILL.md) | Memory & Context | 5/10 | 210 |
| [dynamic-role-assignment-multi-agent](skills/dynamic-role-assignment-multi-agent/SKILL.md) | Multi-Agent Systems | 6/10 | 182 |
| [dziribot-rag-intelligent-conversational](skills/dziribot-rag-intelligent-conversational/SKILL.md) | RAG & Retrieval | 5/10 | 236 |
| [ecco-evidence-driven-causal-reasoning](skills/ecco-evidence-driven-causal-reasoning/SKILL.md) | Efficiency & Optimization | 5/10 | 205 |
| [echo-open-research-platform](skills/echo-open-research-platform/SKILL.md) | Domain-Specific | 6/10 | 207 |
| [echoes-loop-diagnosing-risks](skills/echoes-loop-diagnosing-risks/SKILL.md) | Evaluation & Benchmarking | 5/10 | 395 |
| [effgen-enabling-small-language](skills/effgen-enabling-small-language/SKILL.md) | Agentic Systems | 5/10 | 193 |
| [efficient-table-retrieval-understanding](skills/efficient-table-retrieval-understanding/SKILL.md) | RAG & Retrieval | 4/10 | 178 |
| [eft-cot-multi-agent-chain-of-thought-framework](skills/eft-cot-multi-agent-chain-of-thought-framework/SKILL.md) | Domain-Specific | 3/10 | 300 |
| [egss-entropy-guided-stepwise-scaling](skills/egss-entropy-guided-stepwise-scaling/SKILL.md) | Reasoning & Chain-of-Thought | 4/10 | 164 |
| [eliciting-least-to-most-reasoning-phishing](skills/eliciting-least-to-most-reasoning-phishing/SKILL.md) | Security & Safety | 6/10 | 155 |
| [embodied-task-planning-graph-informed](skills/embodied-task-planning-graph-informed/SKILL.md) | Agentic Systems | 4/10 | 179 |
| [empirical-mcts-continuous-agent-evolution](skills/empirical-mcts-continuous-agent-evolution/SKILL.md) | Reasoning & Chain-of-Thought | 5/10 | 195 |
| [enhancing-mathematical-problem-solving](skills/enhancing-mathematical-problem-solving/SKILL.md) | Reasoning & Chain-of-Thought | 7/10 | 183 |
| [entworld-holistic-environment-benchmark](skills/entworld-holistic-environment-benchmark/SKILL.md) | Evaluation & Benchmarking | 5/10 | 158 |
| [environment-in-the-loop-rethinking-code-migration-](skills/environment-in-the-loop-rethinking-code-migration/SKILL.md) | Code & Software Engineering | 8/10 | 162 |
| [epistemic-context-learning-building](skills/epistemic-context-learning-building/SKILL.md) | Multi-Agent Systems | 5/10 | 210 |
| [error-taxonomy-guided-prompt-optimization](skills/error-taxonomy-guided-prompt-optimization/SKILL.md) | Prompt Engineering | 7/10 | 162 |
| [es-memeval-benchmarking-conversational-agents](skills/es-memeval-benchmarking-conversational-agents/SKILL.md) | Evaluation & Benchmarking | 5/10 | 226 |
| [evaluating-achieving-controllable-code](skills/evaluating-achieving-controllable-code/SKILL.md) | Code & Software Engineering | 7/10 | 197 |
| [evaluating-enhancing-vulnerability-reasoning](skills/evaluating-enhancing-vulnerability-reasoning/SKILL.md) | Security & Safety | 7/10 | 211 |
| [evaluating-kubernetes-performance-genai](skills/evaluating-kubernetes-performance-genai/SKILL.md) | Domain-Specific | 5/10 | 217 |
| [evaluating-retrievalaugmented-generation-variants](skills/evaluating-retrievalaugmented-generation-variants/SKILL.md) | RAG & Retrieval | 5/10 | 223 |
| [evaluating-social-bias-rag](skills/evaluating-social-bias-rag/SKILL.md) | Evaluation & Benchmarking | 4/10 | 212 |
| [evaluating-they-not-know](skills/evaluating-they-not-know/SKILL.md) | Evaluation & Benchmarking | 5/10 | 185 |
| [evaluation-entity-matching-recommender](skills/evaluation-entity-matching-recommender/SKILL.md) | Evaluation & Benchmarking | 5/10 | 182 |
| [evaluation-legal-applications-challenges](skills/evaluation-legal-applications-challenges/SKILL.md) | Evaluation & Benchmarking | 5/10 | 171 |
| [evaluation-oncotimia-system-supporting](skills/evaluation-oncotimia-system-supporting/SKILL.md) | RAG & Retrieval | 5/10 | 213 |
| [event-vstream-event-driven-real-time-understanding](skills/event-vstream-event-driven-real-time-understanding/SKILL.md) | Multimodal | 4/10 | 254 |
| [eventcast-hybrid-demand-forecasting](skills/eventcast-hybrid-demand-forecasting/SKILL.md) | Domain-Specific | 5/10 | 228 |
| [evermembench-benchmarking-long-term-interactive](skills/evermembench-benchmarking-long-term-interactive/SKILL.md) | Evaluation & Benchmarking | 5/10 | 182 |
| [evocodebench-human-performance-benchmark-self-evol](skills/evocodebench-human-performance-benchmark-self-evol/SKILL.md) | Code & Software Engineering | 7/10 | 174 |
| [evoconfig-self-evolving-multi-agent-systems](skills/evoconfig-self-evolving-multi-agent-systems/SKILL.md) | Multi-Agent Systems | 6/10 | 194 |
| [evolve-evolutionary-search-llm-based](skills/evolve-evolutionary-search-llm-based/SKILL.md) | Domain-Specific | 5/10 | 174 |
| [evolving-tool-user-creator](skills/evolving-tool-user-creator/SKILL.md) | Agentic Systems | 5/10 | 181 |
| [experience-driven-multi-agent-systems-training-fre](skills/experience-driven-multi-agent-systems-training-fre/SKILL.md) | Multi-Agent Systems | 5/10 | 168 |
| [explicit-multi-head-attention-inter-head](skills/explicit-multi-head-attention-inter-head/SKILL.md) | Fine-tuning & Training | 4/10 | 217 |
| [exploring-reasoning-reward-agents](skills/exploring-reasoning-reward-agents/SKILL.md) | Evaluation & Benchmarking | 5/10 | 219 |
| [extracting-recurring-vulnerabilities-black-box](skills/extracting-recurring-vulnerabilities-black-box/SKILL.md) | Security & Safety | 6/10 | 188 |
| [fademem-biologically-inspired-forgetting-agent](skills/fademem-biologically-inspired-forgetting-agent/SKILL.md) | Memory & Context | 7/10 | 222 |
| [failure-aware-enhancements-code-generation](skills/failure-aware-enhancements-code-generation/SKILL.md) | Code & Software Engineering | 6/10 | 191 |
| [farm-field-aware-resolution-intelligent](skills/farm-field-aware-resolution-intelligent/SKILL.md) | Agentic Systems | 4/10 | 187 |
| [fat-cat-document-driven-metacognitive-multi-agent](skills/fat-cat-document-driven-metacognitive-multi-agent/SKILL.md) | Agentic Systems | 5/10 | 225 |
| [featurebench-benchmarking-agentic-coding](skills/featurebench-benchmarking-agentic-coding/SKILL.md) | Evaluation & Benchmarking | 5/10 | 178 |
| [fin-rate-real-world-financial-analytics](skills/fin-rate-real-world-financial-analytics/SKILL.md) | Domain-Specific | 5/10 | 182 |
| [fine-tuning-gpt-5-gpu-kernel](skills/fine-tuning-gpt-5-gpu-kernel/SKILL.md) | Efficiency & Optimization | 5/10 | 259 |
| [flyaoc-evaluating-agentic-ontology](skills/flyaoc-evaluating-agentic-ontology/SKILL.md) | Multi-Agent Systems | 5/10 | 184 |
| [fmbench-adaptive-output-formatting](skills/fmbench-adaptive-output-formatting/SKILL.md) | NLP & Text | 5/10 | 224 |
| [following-dragons-code-review-guided](skills/following-dragons-code-review-guided/SKILL.md) | Security & Safety | 6/10 | 158 |
| [fraudshield-knowledge-graph-empowered](skills/fraudshield-knowledge-graph-empowered/SKILL.md) | Security & Safety | 5/10 | 269 |
| [from-assistant-double-agent](skills/from-assistant-double-agent/SKILL.md) | Security & Safety | 7/10 | 230 |
| [from-assumptions-actions-turning](skills/from-assumptions-actions-turning/SKILL.md) | Multi-Agent Systems | 5/10 | 242 |
| [from-code-centric-concept-centric-teaching](skills/from-code-centric-concept-centric-teaching/SKILL.md) | Domain-Specific | 5/10 | 269 |
| [from-detection-prevention-explaining](skills/from-detection-prevention-explaining/SKILL.md) | Security & Safety | 7/10 | 265 |
| [from-features-actions-explainability](skills/from-features-actions-explainability/SKILL.md) | Explainability | 5/10 | 207 |
| [from-gameplay-traces-game](skills/from-gameplay-traces-game/SKILL.md) | Domain-Specific | 5/10 | 211 |
| [from-helpfulness-toxic-proactivity](skills/from-helpfulness-toxic-proactivity/SKILL.md) | Evaluation & Benchmarking | 4/10 | 194 |
| [from-passive-metric-active](skills/from-passive-metric-active/SKILL.md) | Agentic Systems | 5/10 | 268 |
| [from-pragmas-partners-symbiotic](skills/from-pragmas-partners-symbiotic/SKILL.md) | Domain-Specific | 5/10 | 186 |
| [from-prompt-response-goal-directed-systems](skills/from-prompt-response-goal-directed-systems/SKILL.md) | Agentic Systems | 6/10 | 177 |
| [from-sparse-decisions-dense](skills/from-sparse-decisions-dense/SKILL.md) | Security & Safety | 4/10 | 261 |
| [from-task-solving-robust](skills/from-task-solving-robust/SKILL.md) | Agentic Systems | 7/10 | 199 |
| [fs-researcher-test-time-scaling-long-horizon](skills/fs-researcher-test-time-scaling-long-horizon/SKILL.md) | Agentic Systems | 7/10 | 221 |
| [fullstack-agent-enhancing-agentic-fullstack](skills/fullstack-agent-enhancing-agentic-fullstack/SKILL.md) | Code & Software Engineering | 7/10 | 146 |
| [funny-or-persuasive-but](skills/funny-or-persuasive-but/SKILL.md) | Prompt Engineering | 6/10 | 178 |
| [funprm-function-as-step-process-reward](skills/funprm-function-as-step-process-reward/SKILL.md) | Reasoning & Chain-of-Thought | 7/10 | 251 |
| [gamedevbench-evaluating-agentic-capabilities](skills/gamedevbench-evaluating-agentic-capabilities/SKILL.md) | Domain-Specific | 6/10 | 187 |
| [gamms-graph-based-adversarial](skills/gamms-graph-based-adversarial/SKILL.md) | Domain-Specific | 5/10 | 305 |
| [gdcnet-generative-discrepancy-comparison](skills/gdcnet-generative-discrepancy-comparison/SKILL.md) | Multimodal | 5/10 | 180 |
| [gender-race-bias-consumer](skills/gender-race-bias-consumer/SKILL.md) | Evaluation & Benchmarking | 5/10 | 255 |
| [generating-data-driven-reasoning-rubrics](skills/generating-data-driven-reasoning-rubrics/SKILL.md) | Evaluation & Benchmarking | 6/10 | 170 |
| [generative-ontology-structured-knowledge](skills/generative-ontology-structured-knowledge/SKILL.md) | Knowledge Graphs | 6/10 | 222 |
| [gflowpo-generative-flow-network](skills/gflowpo-generative-flow-network/SKILL.md) | Prompt Engineering | 7/10 | 171 |
| [gisa-benchmark-general-information-seeking](skills/gisa-benchmark-general-information-seeking/SKILL.md) | Agentic Systems | 5/10 | 162 |
| [gradingattack-attacking-short-answer](skills/gradingattack-attacking-short-answer/SKILL.md) | Security & Safety | 5/10 | 243 |
| [graph-anchored-knowledge-indexing-retrieval-augmen](skills/graph-anchored-knowledge-indexing-retrieval-augmen/SKILL.md) | RAG & Retrieval | 6/10 | 221 |
| [graph-based-agent-memory-taxonomy](skills/graph-based-agent-memory-taxonomy/SKILL.md) | Knowledge Graphs | 6/10 | 279 |
| [graphagents-knowledge-graph-guided-agentic](skills/graphagents-knowledge-graph-guided-agentic/SKILL.md) | Knowledge Graphs | 5/10 | 185 |
| [graphseek-next-generation-graph-analytics](skills/graphseek-next-generation-graph-analytics/SKILL.md) | Knowledge Graphs | 6/10 | 151 |
| [greprag-empirical-study-optimization](skills/greprag-empirical-study-optimization/SKILL.md) | RAG & Retrieval | 8/10 | 190 |
| [grounding-generative-planners-verifiable](skills/grounding-generative-planners-verifiable/SKILL.md) | Knowledge Graphs | 5/10 | 176 |
| [guideai-real-time-personalized-learning](skills/guideai-real-time-personalized-learning/SKILL.md) | Domain-Specific | 5/10 | 276 |
| [gutenocr-grounded-vision-language-front-end](skills/gutenocr-grounded-vision-language-front-end/SKILL.md) | Multimodal | 4/10 | 210 |
| [haif-human-ai-integration-framework](skills/haif-human-ai-integration-framework/SKILL.md) | Agentic Systems | 5/10 | 206 |
| [hallucination-resistant-security-planning](skills/hallucination-resistant-security-planning/SKILL.md) | Security & Safety | 5/10 | 259 |
| [halluverse-m3-multitask-multilingual-benchmark-hal](skills/halluverse-m3-multitask-multilingual-benchmark-hal/SKILL.md) | Evaluation & Benchmarking | 4/10 | 148 |
| [harmoni-multimodal-personalization-multi-user](skills/harmoni-multimodal-personalization-multi-user/SKILL.md) | Memory & Context | 6/10 | 197 |
| [harnessing-precision-querying-retrieval-augmented](skills/harnessing-precision-querying-retrieval-augmented/SKILL.md) | Data Processing | 6/10 | 163 |
| [helios-hierarchical-graph-abstraction](skills/helios-hierarchical-graph-abstraction/SKILL.md) | Security & Safety | 6/10 | 204 |
| [helm-human-centered-evaluation-framework](skills/helm-human-centered-evaluation-framework/SKILL.md) | Evaluation & Benchmarking | 5/10 | 244 |
| [hidden-licensing-risks-llmware](skills/hidden-licensing-risks-llmware/SKILL.md) | Security & Safety | 6/10 | 182 |
| [history-guided-iterative-visual-reasoning](skills/history-guided-iterative-visual-reasoning/SKILL.md) | Reasoning & Chain-of-Thought | 6/10 | 170 |
| [how-few-shot-demonstrations-affect](skills/how-few-shot-demonstrations-affect/SKILL.md) | Prompt Engineering | 6/10 | 199 |
| [how-information-access-affect](skills/how-information-access-affect/SKILL.md) | Security & Safety | 5/10 | 194 |
| [how-much-reasoning-retrieval-augmented](skills/how-much-reasoning-retrieval-augmented/SKILL.md) | Evaluation & Benchmarking | 5/10 | 178 |
| [how-personalized-memory-shape](skills/how-personalized-memory-shape/SKILL.md) | Memory & Context | 5/10 | 216 |
| [how-well-open-sourced](skills/how-well-open-sourced/SKILL.md) | Evaluation & Benchmarking | 4/10 | 232 |
| [hqp-sensitivity-aware-hybrid-quantization](skills/hqp-sensitivity-aware-hybrid-quantization/SKILL.md) | Efficiency & Optimization | 5/10 | 189 |
| [hugrag-hierarchical-causal-knowledge](skills/hugrag-hierarchical-causal-knowledge/SKILL.md) | Knowledge Graphs | 6/10 | 168 |
| [human-aligned-enhancement-programming-answers](skills/human-aligned-enhancement-programming-answers/SKILL.md) | NLP & Text | 5/10 | 258 |
| [humans-welcome-observe-first-look](skills/humans-welcome-observe-first-look/SKILL.md) | Evaluation & Benchmarking | 4/10 | 191 |
| [hunt-instead-wait-evaluating](skills/hunt-instead-wait-evaluating/SKILL.md) | Data Processing | 7/10 | 222 |
| [hybrid-supervised-llm-pipeline-actionable-suggesti](skills/hybrid-supervised-llm-pipeline-actionable-suggesti/SKILL.md) | NLP & Text | 4/10 | 193 |
| [hyperoffload-graph-driven-hierarchical-memory](skills/hyperoffload-graph-driven-hierarchical-memory/SKILL.md) | Efficiency & Optimization | 4/10 | 226 |
| [ic-eo-interpretable-code-based-assistant](skills/ic-eo-interpretable-code-based-assistant/SKILL.md) | Domain-Specific | 5/10 | 215 |
| [icl-evader-zero-query-black-box-evasion](skills/icl-evader-zero-query-black-box-evasion/SKILL.md) | Security & Safety | 6/10 | 251 |
| [icon-intent-context-coupling-multi-turn](skills/icon-intent-context-coupling-multi-turn/SKILL.md) | Evaluation & Benchmarking | 5/10 | 243 |
| [ide-bench-evaluating-as-ide](skills/ide-bench-evaluating-as-ide/SKILL.md) | Code & Software Engineering | 8/10 | 149 |
| [identifying-adversary-tactics-techniques](skills/identifying-adversary-tactics-techniques/SKILL.md) | Security & Safety | 6/10 | 198 |
| [identifying-concurrency-bug-reports](skills/identifying-concurrency-bug-reports/SKILL.md) | Code & Software Engineering | 6/10 | 185 |
| [iesr-mcts-based-modular-reasoning](skills/iesr-mcts-based-modular-reasoning/SKILL.md) | Reasoning & Chain-of-Thought | 6/10 | 242 |
| [infa-guard-mitigating-malicious-propagation](skills/infa-guard-mitigating-malicious-propagation/SKILL.md) | Multi-Agent Systems | 5/10 | 262 |
| [internet-agentic-ai-incentive-compatible](skills/internet-agentic-ai-incentive-compatible/SKILL.md) | Multi-Agent Systems | 4/10 | 208 |
| [interpreting-agentic-systems-beyond](skills/interpreting-agentic-systems-beyond/SKILL.md) | Explainability | 6/10 | 329 |
| [interpreting-controlling-behavior-constitutions](skills/interpreting-controlling-behavior-constitutions/SKILL.md) | Prompt Engineering | 5/10 | 182 |
| [isd-agent-bench-comprehensive-benchmark-evaluating](skills/isd-agent-bench-comprehensive-benchmark-evaluating/SKILL.md) | Evaluation & Benchmarking | 4/10 | 210 |
| [issueguard-real-time-secret-leak](skills/issueguard-real-time-secret-leak/SKILL.md) | Security & Safety | 8/10 | 268 |
| [jaf-judge-agent-forest](skills/jaf-judge-agent-forest/SKILL.md) | Evaluation & Benchmarking | 6/10 | 213 |
| [jailbreaks-vision-multimodal-reasoning](skills/jailbreaks-vision-multimodal-reasoning/SKILL.md) | Security & Safety | 5/10 | 250 |
| [jobresqa-benchmark-machine-reading](skills/jobresqa-benchmark-machine-reading/SKILL.md) | Evaluation & Benchmarking | 4/10 | 152 |
| [just-ask-curious-code](skills/just-ask-curious-code/SKILL.md) | Security & Safety | 5/10 | 224 |
| [just-in-time-reinforcement-learning-continual](skills/just-in-time-reinforcement-learning-continual/SKILL.md) | Agentic Systems | 5/10 | 203 |
| [kg-craft-knowledge-graph-based-contrastive](skills/kg-craft-knowledge-graph-based-contrastive/SKILL.md) | Knowledge Graphs | 5/10 | 197 |
| [knowledge-restoration-driven-prompt-optimization](skills/knowledge-restoration-driven-prompt-optimization/SKILL.md) | Prompt Engineering | 5/10 | 211 |
| [koral-knowledge-graph-guided](skills/koral-knowledge-graph-guided/SKILL.md) | Knowledge Graphs | 5/10 | 247 |
| [krone-hierarchical-modular-log](skills/krone-hierarchical-modular-log/SKILL.md) | Data Processing | 5/10 | 240 |
| [large-geolocation-extraction-humanitarian](skills/large-geolocation-extraction-humanitarian/SKILL.md) | NLP & Text | 5/10 | 213 |
| [large-model-powered-evolutionary-code](skills/large-model-powered-evolutionary-code/SKILL.md) | Efficiency & Optimization | 7/10 | 186 |
| [large-reasoning-failures](skills/large-reasoning-failures/SKILL.md) | Reasoning & Chain-of-Thought | 6/10 | 218 |
| [large-scale-multidimensional-knowledge-profiling](skills/large-scale-multidimensional-knowledge-profiling/SKILL.md) | Data Processing | 5/10 | 246 |
| [lata-tool-llm-assisted-translation](skills/lata-tool-llm-assisted-translation/SKILL.md) | NLP & Text | 5/10 | 232 |
| [latent-chain-of-thought-as-planning](skills/latent-chain-of-thought-as-planning/SKILL.md) | Reasoning & Chain-of-Thought | 5/10 | 200 |
| [layer-wise-lora-fine-tuning-similarity](skills/layer-wise-lora-fine-tuning-similarity/SKILL.md) | Fine-tuning & Training | 5/10 | 221 |
| [learning-compose-cross-domain-agentic](skills/learning-compose-cross-domain-agentic/SKILL.md) | Agentic Systems | 5/10 | 159 |
| [learning-irrecoverable-error-localized-policy](skills/learning-irrecoverable-error-localized-policy/SKILL.md) | Agentic Systems | 6/10 | 175 |
| [learning-rate-matters-vanilla](skills/learning-rate-matters-vanilla/SKILL.md) | Fine-tuning & Training | 4/10 | 203 |
| [learning-reason-faithfully-step-level](skills/learning-reason-faithfully-step-level/SKILL.md) | Reasoning & Chain-of-Thought | 5/10 | 173 |
| [legalmalr-multi-agent-query-understanding](skills/legalmalr-multi-agent-query-understanding/SKILL.md) | RAG & Retrieval | 6/10 | 168 |
| [lemon-agent-technical-report](skills/lemon-agent-technical-report/SKILL.md) | Multi-Agent Systems | 6/10 | 186 |
| [less-noise-more-voice](skills/less-noise-more-voice/SKILL.md) | Prompt Engineering | 5/10 | 236 |
| [leveraging-data-say-no](skills/leveraging-data-say-no/SKILL.md) | Multimodal | 4/10 | 194 |
| [leveraging-turkish-skill-extraction](skills/leveraging-turkish-skill-extraction/SKILL.md) | NLP & Text | 5/10 | 198 |
| [lhaw-controllable-underspecification-long-horizon](skills/lhaw-controllable-underspecification-long-horizon/SKILL.md) | Prompt Engineering | 5/10 | 170 |
| [linglanmidian-systematic-evaluation-tcm](skills/linglanmidian-systematic-evaluation-tcm/SKILL.md) | Evaluation & Benchmarking | 5/10 | 245 |
| [linguistagent-a-reflective-multimodel](skills/linguistagent-a-reflective-multimodel/SKILL.md) | NLP & Text | 6/10 | 241 |
| [lingxidiagbench-multi-agent-framework-benchmarking](skills/lingxidiagbench-multi-agent-framework-benchmarking/SKILL.md) | Evaluation & Benchmarking | 5/10 | 216 |
| [live-evo-online-evolution-agentic](skills/live-evo-online-evolution-agentic/SKILL.md) | Memory & Context | 6/10 | 204 |
| [livemedbench-contamination-free-medical-benchmark](skills/livemedbench-contamination-free-medical-benchmark/SKILL.md) | Evaluation & Benchmarking | 5/10 | 296 |
| [livibench-omnimodal-benchmark-interactive](skills/livibench-omnimodal-benchmark-interactive/SKILL.md) | Evaluation & Benchmarking | 4/10 | 238 |
| [llama-31-foundationai-securityllm-reasoning-8b-tec](skills/llama-31-foundationai-securityllm-reasoning-8b-tec/SKILL.md) | Security & Safety | 5/10 | 251 |
| [llm-assisted-logic-rule-learning](skills/llm-assisted-logic-rule-learning/SKILL.md) | Data Processing | 6/10 | 181 |
| [llm-based-sql-generation-prompting](skills/llm-based-sql-generation-prompting/SKILL.md) | Prompt Engineering | 7/10 | 174 |
| [llm-fsm-scaling-finite-state-reasoning](skills/llm-fsm-scaling-finite-state-reasoning/SKILL.md) | Domain-Specific | 7/10 | 265 |
| [llm-in-sandbox-elicits-general-agentic](skills/llm-in-sandbox-elicits-general-agentic/SKILL.md) | Agentic Systems | 7/10 | 198 |
| [llm-not-all-you](skills/llm-not-all-you/SKILL.md) | Domain-Specific | 5/10 | 187 |
| [llm-prompt-evaluation-educational](skills/llm-prompt-evaluation-educational/SKILL.md) | Prompt Engineering | 5/10 | 223 |
| [llms-as-orchestrators-constraint-compliant](skills/llms-as-orchestrators-constraint-compliant/SKILL.md) | Multi-Agent Systems | 5/10 | 186 |
| [loca-bench-benchmarking-agents-under](skills/loca-bench-benchmarking-agents-under/SKILL.md) | Memory & Context | 7/10 | 169 |
| [localv-exploiting-information-locality](skills/localv-exploiting-information-locality/SKILL.md) | Domain-Specific | 6/10 | 138 |
| [locomo-plus-beyond-factual-cognitive-memory](skills/locomo-plus-beyond-factual-cognitive-memory/SKILL.md) | Memory & Context | 4/10 | 216 |
| [logicscore-fine-grained-logic-evaluation](skills/logicscore-fine-grained-logic-evaluation/SKILL.md) | Evaluation & Benchmarking | 5/10 | 185 |
| [logsieve-task-aware-ci-log](skills/logsieve-task-aware-ci-log/SKILL.md) | Efficiency & Optimization | 7/10 | 167 |
| [longcat-flash-thinking-2601-technical-report](skills/longcat-flash-thinking-2601-technical-report/SKILL.md) | Agentic Systems | 5/10 | 311 |
| [lps-bench-benchmarking-safety-awareness](skills/lps-bench-benchmarking-safety-awareness/SKILL.md) | Security & Safety | 5/10 | 191 |
| [made-benchmark-environments-closed-loop](skills/made-benchmark-environments-closed-loop/SKILL.md) | Evaluation & Benchmarking | 5/10 | 144 |
| [magellan-autonomous-discovery-compiler](skills/magellan-autonomous-discovery-compiler/SKILL.md) | Domain-Specific | 3/10 | 158 |
| [malicious-agent-skills-wild](skills/malicious-agent-skills-wild/SKILL.md) | Security & Safety | 7/10 | 264 |
| [malicious-repurposing-open-science](skills/malicious-repurposing-open-science/SKILL.md) | Security & Safety | 4/10 | 212 |
| [markovscale-optimal-sequential-scaling](skills/markovscale-optimal-sequential-scaling/SKILL.md) | Efficiency & Optimization | 6/10 | 193 |
| [marti-mars2-scaling-multi-agent-self-search-reinfo](skills/marti-mars2-scaling-multi-agent-self-search-reinfo/SKILL.md) | Multi-Agent Systems | 7/10 | 204 |
| [mas-prove-understanding-process-verification](skills/mas-prove-understanding-process-verification/SKILL.md) | Multi-Agent Systems | 5/10 | 237 |
| [masalbench-benchmark-contextual-cross-cultural](skills/masalbench-benchmark-contextual-cross-cultural/SKILL.md) | Evaluation & Benchmarking | 5/10 | 186 |
| [mascot-multi-agent-socio-collaborative-companion](skills/mascot-multi-agent-socio-collaborative-companion/SKILL.md) | Multi-Agent Systems | 5/10 | 244 |
| [mata-multiagent-framework-for](skills/mata-multiagent-framework-for/SKILL.md) | Data Processing | 6/10 | 167 |
| [mathliblemma-folklore-lemma-generation](skills/mathliblemma-folklore-lemma-generation/SKILL.md) | Domain-Specific | 4/10 | 181 |
| [mcp-atlas-large-scale-benchmark-tool-use](skills/mcp-atlas-large-scale-benchmark-tool-use/SKILL.md) | Evaluation & Benchmarking | 5/10 | 210 |
| [mdl-unified-multi-distribution-learner](skills/mdl-unified-multi-distribution-learner/SKILL.md) | Domain-Specific | 4/10 | 177 |
| [medbeads-agent-native-immutable-data](skills/medbeads-agent-native-immutable-data/SKILL.md) | Domain-Specific | 5/10 | 182 |
| [medspeak-knowledge-graph-aided-asr](skills/medspeak-knowledge-graph-aided-asr/SKILL.md) | Knowledge Graphs | 5/10 | 262 |
| [medverse-reliable-medical-reasoning](skills/medverse-reliable-medical-reasoning/SKILL.md) | Domain-Specific | 5/10 | 216 |
| [memcast-memory-driven-time-series](skills/memcast-memory-driven-time-series/SKILL.md) | Memory & Context | 5/10 | 196 |
| [mempot-defending-against-memory](skills/mempot-defending-against-memory/SKILL.md) | Security & Safety | 5/10 | 246 |
| [menvagent-scalable-polyglot-environment](skills/menvagent-scalable-polyglot-environment/SKILL.md) | Code & Software Engineering | 7/10 | 188 |
| [mermaid-memory-enhanced-retrieval-reasoning](skills/mermaid-memory-enhanced-retrieval-reasoning/SKILL.md) | RAG & Retrieval | 5/10 | 189 |
| [metagen-self-evolving-roles-topologies](skills/metagen-self-evolving-roles-topologies/SKILL.md) | Multi-Agent Systems | 5/10 | 189 |
| [mhdash-online-platform-benchmarking](skills/mhdash-online-platform-benchmarking/SKILL.md) | Evaluation & Benchmarking | 5/10 | 285 |
| [mirror-multi-agent-framework-iterative](skills/mirror-multi-agent-framework-iterative/SKILL.md) | Domain-Specific | 7/10 | 166 |
| [mitigating-conversational-inertia-multi-turn](skills/mitigating-conversational-inertia-multi-turn/SKILL.md) | Agentic Systems | 6/10 | 236 |
| [mmr-bench-comprehensive-benchmark-multimodal](skills/mmr-bench-comprehensive-benchmark-multimodal/SKILL.md) | Efficiency & Optimization | 5/10 | 175 |
| [moco-one-stop-shop-collaboration](skills/moco-one-stop-shop-collaboration/SKILL.md) | Multi-Agent Systems | 4/10 | 225 |
| [modality-gap-driven-subspace-alignment](skills/modality-gap-driven-subspace-alignment/SKILL.md) | Multimodal | 4/10 | 231 |
| [monte-carlo-tree-search](skills/monte-carlo-tree-search/SKILL.md) | Reasoning & Chain-of-Thought | 7/10 | 186 |
| [more-code-less-reuse](skills/more-code-less-reuse/SKILL.md) | Code & Software Engineering | 7/10 | 187 |
| [more-than-quick-glance](skills/more-than-quick-glance/SKILL.md) | Efficiency & Optimization | 4/10 | 256 |
| [mpib-benchmark-medical-prompt](skills/mpib-benchmark-medical-prompt/SKILL.md) | Evaluation & Benchmarking | 5/10 | 177 |
| [mrag-benchmarking-retrieval-augmented-generation](skills/mrag-benchmarking-retrieval-augmented-generation/SKILL.md) | RAG & Retrieval | 5/10 | 183 |
| [multi-agent-causal-reasoning-system](skills/multi-agent-causal-reasoning-system/SKILL.md) | Multi-Agent Systems | 5/10 | 225 |
| [multi-agent-constraint-factorization-reveals](skills/multi-agent-constraint-factorization-reveals/SKILL.md) | Multi-Agent Systems | 6/10 | 157 |
| [multi-agent-end-to-end-vulnerability-management](skills/multi-agent-end-to-end-vulnerability-management/SKILL.md) | Security & Safety | 6/10 | 196 |
| [multi-agent-teams-hold-experts](skills/multi-agent-teams-hold-experts/SKILL.md) | Multi-Agent Systems | 7/10 | 151 |
| [multi-field-tool-retrieval](skills/multi-field-tool-retrieval/SKILL.md) | RAG & Retrieval | 6/10 | 284 |
| [multivis-agent-multi-agent-framework-logic](skills/multivis-agent-multi-agent-framework-logic/SKILL.md) | Multi-Agent Systems | 5/10 | 193 |
| [mulvul-retrieval-augmented-multi-agent-code](skills/mulvul-retrieval-augmented-multi-agent-code/SKILL.md) | Security & Safety | 6/10 | 204 |
| [naamse-framework-evolutionary-security](skills/naamse-framework-evolutionary-security/SKILL.md) | Security & Safety | 6/10 | 201 |
| [neural-theorem-proving-verification](skills/neural-theorem-proving-verification/SKILL.md) | Code & Software Engineering | 5/10 | 202 |
| [next-gen-captchas-leveraging-cognitive](skills/next-gen-captchas-leveraging-cognitive/SKILL.md) | Security & Safety | 6/10 | 253 |
| [noisy-but-valid-robust](skills/noisy-but-valid-robust/SKILL.md) | Evaluation & Benchmarking | 6/10 | 242 |
| [now-you-hear-me](skills/now-you-hear-me/SKILL.md) | Security & Safety | 5/10 | 311 |
| [odysseyarena-benchmarking-long-horizon-active](skills/odysseyarena-benchmarking-long-horizon-active/SKILL.md) | Evaluation & Benchmarking | 4/10 | 176 |
| [omni-rrm-advancing-omni-reward](skills/omni-rrm-advancing-omni-reward/SKILL.md) | Evaluation & Benchmarking | 5/10 | 180 |
| [omnicode-benchmark-evaluating-software](skills/omnicode-benchmark-evaluating-software/SKILL.md) | Evaluation & Benchmarking | 6/10 | 199 |
| [omnireview-large-scale-benchmark-llm-enhanced](skills/omnireview-large-scale-benchmark-llm-enhanced/SKILL.md) | Domain-Specific | 4/10 | 204 |
| [on-impact-agentsmd-files](skills/on-impact-agentsmd-files/SKILL.md) | Efficiency & Optimization | 9/10 | 196 |
| [on-impact-code-comments](skills/on-impact-code-comments/SKILL.md) | Code & Software Engineering | 7/10 | 219 |
| [on-uncertainty-model-based-multi-agent](skills/on-uncertainty-model-based-multi-agent/SKILL.md) | Multi-Agent Systems | 4/10 | 184 |
| [on-use-generate-dataset](skills/on-use-generate-dataset/SKILL.md) | Evaluation & Benchmarking | 6/10 | 195 |
| [on-use-support-conduction](skills/on-use-support-conduction/SKILL.md) | Data Processing | 5/10 | 167 |
| [ontology-to-tools-compilation-executable-semantic-](skills/ontology-to-tools-compilation-executable-semantic/SKILL.md) | Knowledge Graphs | 5/10 | 192 |
| [open-tutorai-open-source-platform](skills/open-tutorai-open-source-platform/SKILL.md) | Domain-Specific | 5/10 | 266 |
| [openguandan-large-scale-imperfect-information](skills/openguandan-large-scale-imperfect-information/SKILL.md) | Domain-Specific | 5/10 | 366 |
| [opinf-llm-parametric-pde-solving](skills/opinf-llm-parametric-pde-solving/SKILL.md) | Domain-Specific | 6/10 | 200 |
| [opportunities-aiml-rubin-lsst](skills/opportunities-aiml-rubin-lsst/SKILL.md) | Domain-Specific | 5/10 | 249 |
| [optimal-turkish-subword-strategies](skills/optimal-turkish-subword-strategies/SKILL.md) | NLP & Text | 4/10 | 253 |
| [optimizing-small-sample-experience-learning-llm-ba](skills/optimizing-small-sample-experience-learning-llm-ba/SKILL.md) | NLP & Text | 5/10 | 195 |
| [orthogonal-hierarchical-decomposition-structure-aw](skills/orthogonal-hierarchical-decomposition-structure-aw/SKILL.md) | Data Processing | 6/10 | 231 |
| [pabu-progress-aware-belief-update](skills/pabu-progress-aware-belief-update/SKILL.md) | Agentic Systems | 7/10 | 210 |
| [pamas-self-adaptive-multi-agent-system](skills/pamas-self-adaptive-multi-agent-system/SKILL.md) | Multi-Agent Systems | 5/10 | 162 |
| [paperbanana-automating-academic-illustration](skills/paperbanana-automating-academic-illustration/SKILL.md) | Multimodal | 4/10 | 197 |
| [papersearchqa-learning-search-reason](skills/papersearchqa-learning-search-reason/SKILL.md) | RAG & Retrieval | 5/10 | 233 |
| [parse-open-domain-reasoning-question](skills/parse-open-domain-reasoning-question/SKILL.md) | Evaluation & Benchmarking | 4/10 | 220 |
| [patch-to-poc-systematic-study-agentic](skills/patch-to-poc-systematic-study-agentic/SKILL.md) | Security & Safety | 5/10 | 172 |
| [pathwise-planning-world-automated](skills/pathwise-planning-world-automated/SKILL.md) | Agentic Systems | 6/10 | 165 |
| [pcbschemagen-constraint-guided-schematic-design](skills/pcbschemagen-constraint-guided-schematic-design/SKILL.md) | Domain-Specific | 5/10 | 209 |
| [pearl-plan-exploration-adaptive](skills/pearl-plan-exploration-adaptive/SKILL.md) | Agentic Systems | 7/10 | 172 |
| [pearl-prototype-enhanced-alignment-label-efficient](skills/pearl-prototype-enhanced-alignment-label-efficient/SKILL.md) | RAG & Retrieval | 5/10 | 222 |
| [perfguard-performance-aware-agent-visual](skills/perfguard-performance-aware-agent-visual/SKILL.md) | Agentic Systems | 4/10 | 227 |
| [persona-jailbreaking](skills/persona-jailbreaking/SKILL.md) | Security & Safety | 5/10 | 301 |
| [personality-as-relational-infrastructure](skills/personality-as-relational-infrastructure/SKILL.md) | Prompt Engineering | 5/10 | 252 |
| [physical-prompt-injection-attacks](skills/physical-prompt-injection-attacks/SKILL.md) | Security & Safety | 5/10 | 291 |
| [planner-auditor-twin-agentic-discharge](skills/planner-auditor-twin-agentic-discharge/SKILL.md) | Agentic Systems | 6/10 | 189 |
| [polarmem-training-free-polarized-latent](skills/polarmem-training-free-polarized-latent/SKILL.md) | Memory & Context | 5/10 | 183 |
| [pope-learning-reason-hard](skills/pope-learning-reason-hard/SKILL.md) | Reasoning & Chain-of-Thought | 5/10 | 132 |
| [precise-reducing-bias-evaluations](skills/precise-reducing-bias-evaluations/SKILL.md) | Evaluation & Benchmarking | 6/10 | 212 |
| [precision-practice-knowledge-guided](skills/precision-practice-knowledge-guided/SKILL.md) | Code & Software Engineering | 7/10 | 298 |
| [predicting-improving-test-time-scaling](skills/predicting-improving-test-time-scaling/SKILL.md) | Efficiency & Optimization | 5/10 | 209 |
| [predicting-intermittent-job-failure](skills/predicting-intermittent-job-failure/SKILL.md) | Code & Software Engineering | 4/10 | 280 |
| [predictive-coding-information-bottleneck](skills/predictive-coding-information-bottleneck/SKILL.md) | Evaluation & Benchmarking | 4/10 | 238 |
| [prism-principled-framework-multi-agent](skills/prism-principled-framework-multi-agent/SKILL.md) | Reasoning & Chain-of-Thought | 6/10 | 138 |
| [prism-xr-empowering-privacy-aware-xr](skills/prism-xr-empowering-privacy-aware-xr/SKILL.md) | Security & Safety | 5/10 | 301 |
| [privacy-collapse-benign-fine-tuning](skills/privacy-collapse-benign-fine-tuning/SKILL.md) | Fine-tuning & Training | 4/10 | 195 |
| [proact-agentic-lookahead-interactive](skills/proact-agentic-lookahead-interactive/SKILL.md) | Reasoning & Chain-of-Thought | 6/10 | 241 |
| [probing-knowledge-boundary-interactive](skills/probing-knowledge-boundary-interactive/SKILL.md) | Prompt Engineering | 5/10 | 181 |
| [profinfer-ebpf-based-fine-grained-inference](skills/profinfer-ebpf-based-fine-grained-inference/SKILL.md) | Efficiency & Optimization | 5/10 | 289 |
| [prompt-driven-development-claude](skills/prompt-driven-development-claude/SKILL.md) | Prompt Engineering | 6/10 | 188 |
| [prompt-injection-attacks-agentic](skills/prompt-injection-attacks-agentic/SKILL.md) | Security & Safety | 8/10 | 207 |
| [proopf-benchmarking-improving-professional-grade](skills/proopf-benchmarking-improving-professional-grade/SKILL.md) | Domain-Specific | 5/10 | 218 |
| [protean-compiler-agile-framework](skills/protean-compiler-agile-framework/SKILL.md) | Efficiency & Optimization | 4/10 | 149 |
| [proxywar-dynamic-assessment-of](skills/proxywar-dynamic-assessment-of/SKILL.md) | Evaluation & Benchmarking | 4/10 | 197 |
| [pruning-minimal-reasoning-graphs](skills/pruning-minimal-reasoning-graphs/SKILL.md) | Knowledge Graphs | 5/10 | 154 |
| [puda-private-user-dataset](skills/puda-private-user-dataset/SKILL.md) | Security & Safety | 5/10 | 213 |
| [pull-requests-as-training](skills/pull-requests-as-training/SKILL.md) | Code & Software Engineering | 7/10 | 199 |
| [qrs-rule-synthesizing-neuro-symbolic-triad](skills/qrs-rule-synthesizing-neuro-symbolic-triad/SKILL.md) | Security & Safety | 5/10 | 229 |
| [quasar-universal-autonomous-system](skills/quasar-universal-autonomous-system/SKILL.md) | Domain-Specific | 4/10 | 165 |
| [query-efficient-agentic-graph-extraction](skills/query-efficient-agentic-graph-extraction/SKILL.md) | Security & Safety | 5/10 | 239 |
| [ragturk-best-practices-retrieval](skills/ragturk-best-practices-retrieval/SKILL.md) | RAG & Retrieval | 6/10 | 208 |
| [raicl-retrieval-augmented-in-context-learning](skills/raicl-retrieval-augmented-in-context-learning/SKILL.md) | Multimodal | 5/10 | 228 |
| [ral-bench-benchmarking-application-level-functiona](skills/ral-bench-benchmarking-application-level-functiona/SKILL.md) | Code & Software Engineering | 6/10 | 179 |
| [read-as-human-compressing](skills/read-as-human-compressing/SKILL.md) | Memory & Context | 5/10 | 242 |
| [realhd-high-quality-dataset-robust](skills/realhd-high-quality-dataset-robust/SKILL.md) | Domain-Specific | 4/10 | 230 |
| [realistic-synthetic-household-data](skills/realistic-synthetic-household-data/SKILL.md) | Data Processing | 4/10 | 179 |
| [realsec-bench-benchmark-evaluating-secure](skills/realsec-bench-benchmark-evaluating-secure/SKILL.md) | Security & Safety | 6/10 | 195 |
| [reasoning-augmented-representations-multimodal-ret](skills/reasoning-augmented-representations-multimodal-ret/SKILL.md) | RAG & Retrieval | 5/10 | 223 |
| [reasoning-beyond-literal-cross-style](skills/reasoning-beyond-literal-cross-style/SKILL.md) | Reasoning & Chain-of-Thought | 5/10 | 176 |
| [reasoning-while-asking-transforming](skills/reasoning-while-asking-transforming/SKILL.md) | Reasoning & Chain-of-Thought | 7/10 | 190 |
| [redsage-cybersecurity-generalist](skills/redsage-cybersecurity-generalist/SKILL.md) | Security & Safety | 6/10 | 234 |
| [reducing-costs-proof-synthesis](skills/reducing-costs-proof-synthesis/SKILL.md) | Code & Software Engineering | 5/10 | 247 |
| [refer-agent-collaborative-multi-agent-system](skills/refer-agent-collaborative-multi-agent-system/SKILL.md) | Multi-Agent Systems | 5/10 | 179 |
| [reflect-transparent-principle-guided-reasoning](skills/reflect-transparent-principle-guided-reasoning/SKILL.md) | Prompt Engineering | 6/10 | 185 |
| [refuge-feature-generation-prediction](skills/refuge-feature-generation-prediction/SKILL.md) | Data Processing | 6/10 | 168 |
| [regular-variational-latent-reasoning](skills/regular-variational-latent-reasoning/SKILL.md) | Reasoning & Chain-of-Thought | 5/10 | 236 |
| [reliable-responsible-foundation-comprehensive](skills/reliable-responsible-foundation-comprehensive/SKILL.md) | Security & Safety | 5/10 | 360 |
| [report-nsf-workshop-ai](skills/report-nsf-workshop-ai/SKILL.md) | Domain-Specific | 4/10 | 276 |
| [reprompt-prompt-generation-intelligent](skills/reprompt-prompt-generation-intelligent/SKILL.md) | Prompt Engineering | 7/10 | 191 |
| [research-multi-stage-machine-learning](skills/research-multi-stage-machine-learning/SKILL.md) | RAG & Retrieval | 6/10 | 279 |
| [rethinker-scientific-reasoning-rethinking](skills/rethinker-scientific-reasoning-rethinking/SKILL.md) | Reasoning & Chain-of-Thought | 6/10 | 239 |
| [rethinking-reranker-boundary-aware-evidence](skills/rethinking-reranker-boundary-aware-evidence/SKILL.md) | RAG & Retrieval | 5/10 | 159 |
| [rethinking-scientific-modeling-physically](skills/rethinking-scientific-modeling-physically/SKILL.md) | Domain-Specific | 6/10 | 209 |
| [rethinking-value-agent-generated-tests](skills/rethinking-value-agent-generated-tests/SKILL.md) | Agentic Systems | 7/10 | 163 |
| [revisiting-role-natural-code](skills/revisiting-role-natural-code/SKILL.md) | Code & Software Engineering | 7/10 | 215 |
| [robustexplain-evaluating-robustness-llm-based](skills/robustexplain-evaluating-robustness-llm-based/SKILL.md) | Evaluation & Benchmarking | 5/10 | 211 |
| [roma-recursive-open-meta-agent](skills/roma-recursive-open-meta-agent/SKILL.md) | Agentic Systems | 7/10 | 185 |
| [rubberduckbench-benchmark-ai-coding](skills/rubberduckbench-benchmark-ai-coding/SKILL.md) | Evaluation & Benchmarking | 6/10 | 182 |
| [ruleflow-generating-reusable-program](skills/ruleflow-generating-reusable-program/SKILL.md) | Efficiency & Optimization | 7/10 | 160 |
| [rulesmith-multi-agent-automated-game](skills/rulesmith-multi-agent-automated-game/SKILL.md) | Multi-Agent Systems | 5/10 | 193 |
| [rvb-automating-ai-system](skills/rvb-automating-ai-system/SKILL.md) | Security & Safety | 8/10 | 222 |
| [s3-cot-self-sampled-succinct-reasoning](skills/s3-cot-self-sampled-succinct-reasoning/SKILL.md) | Reasoning & Chain-of-Thought | 4/10 | 201 |
| [safepred-predictive-guardrail-computer-using](skills/safepred-predictive-guardrail-computer-using/SKILL.md) | Security & Safety | 6/10 | 179 |
| [sar-rag-atr-visual-question](skills/sar-rag-atr-visual-question/SKILL.md) | RAG & Retrieval | 5/10 | 378 |
| [scidatacopilot-agentic-data-preparation](skills/scidatacopilot-agentic-data-preparation/SKILL.md) | Data Processing | 6/10 | 240 |
| [scratcheval-multimodal-evaluation-framework](skills/scratcheval-multimodal-evaluation-framework/SKILL.md) | Domain-Specific | 4/10 | 156 |
| [sdr-cir-semantic-debias-retrieval](skills/sdr-cir-semantic-debias-retrieval/SKILL.md) | RAG & Retrieval | 4/10 | 171 |
| [seccodeprm-process-reward-code](skills/seccodeprm-process-reward-code/SKILL.md) | Security & Safety | 7/10 | 201 |
| [secure-code-generation-via](skills/secure-code-generation-via/SKILL.md) | Security & Safety | 7/10 | 254 |
| [self-hinting-enhance-reinforcement-learning](skills/self-hinting-enhance-reinforcement-learning/SKILL.md) | Reasoning & Chain-of-Thought | 5/10 | 174 |
| [semantic-aware-advanced-persistent-threat](skills/semantic-aware-advanced-persistent-threat/SKILL.md) | Security & Safety | 4/10 | 194 |
| [semanticalli-caching-reasoning-not](skills/semanticalli-caching-reasoning-not/SKILL.md) | Efficiency & Optimization | 7/10 | 202 |
| [sere-similarity-based-expert-re-routing](skills/sere-similarity-based-expert-re-routing/SKILL.md) | Efficiency & Optimization | 4/10 | 223 |
| [seta-statistical-fault-attribution](skills/seta-statistical-fault-attribution/SKILL.md) | Evaluation & Benchmarking | 5/10 | 247 |
| [shardmemo-masked-moe-routing](skills/shardmemo-masked-moe-routing/SKILL.md) | Memory & Context | 5/10 | 176 |
| [sifting-noise-comparative-study](skills/sifting-noise-comparative-study/SKILL.md) | Security & Safety | 8/10 | 154 |
| [skillrl-evolving-agents-recursive](skills/skillrl-evolving-agents-recursive/SKILL.md) | Agentic Systems | 5/10 | 184 |
| [small-beautiful-practical-log](skills/small-beautiful-practical-log/SKILL.md) | Data Processing | 5/10 | 191 |
| [smartoracle-agentic-approach](skills/smartoracle-agentic-approach/SKILL.md) | Agentic Systems | 6/10 | 177 |
| [social-catalysts-not-moral](skills/social-catalysts-not-moral/SKILL.md) | Multi-Agent Systems | 5/10 | 180 |
| [socialveil-probing-social-intelligence](skills/socialveil-probing-social-intelligence/SKILL.md) | Evaluation & Benchmarking | 4/10 | 203 |
| [sogptspotter-detecting-chatgpt-generated-answers](skills/sogptspotter-detecting-chatgpt-generated-answers/SKILL.md) | NLP & Text | 4/10 | 200 |
| [solagent-specialized-multi-agent-framework](skills/solagent-specialized-multi-agent-framework/SKILL.md) | Domain-Specific | 7/10 | 193 |
| [sparc-rag-adaptive-sequential-parallel-scaling](skills/sparc-rag-adaptive-sequential-parallel-scaling/SKILL.md) | RAG & Retrieval | 6/10 | 248 |
| [sparc-separating-perception-reasoning](skills/sparc-separating-perception-reasoning/SKILL.md) | Multimodal | 6/10 | 170 |
| [sparseeval-evaluation-sparse-optimization](skills/sparseeval-evaluation-sparse-optimization/SKILL.md) | Evaluation & Benchmarking | 5/10 | 221 |
| [spell-synthesis-programmatic-edits](skills/spell-synthesis-programmatic-edits/SKILL.md) | Code & Software Engineering | 8/10 | 209 |
| [spider-sense-intrinsic-risk-sensing](skills/spider-sense-intrinsic-risk-sensing/SKILL.md) | Security & Safety | 6/10 | 212 |
| [sql-trail-multi-turn-reinforcement-learning](skills/sql-trail-multi-turn-reinforcement-learning/SKILL.md) | Data Processing | 8/10 | 186 |
| [sqlagent-learning-explore-before](skills/sqlagent-learning-explore-before/SKILL.md) | Data Processing | 7/10 | 201 |
| [st-raptor-agentic-system-semi-structured](skills/st-raptor-agentic-system-semi-structured/SKILL.md) | Data Processing | 6/10 | 222 |
| [stalled-biased-confused-uncovering-reasoning](skills/stalled-biased-confused-uncovering-reasoning/SKILL.md) | Reasoning & Chain-of-Thought | 6/10 | 168 |
| [standardizing-longitudinal-radiology-report](skills/standardizing-longitudinal-radiology-report/SKILL.md) | NLP & Text | 5/10 | 230 |
| [state-art-llm-enabled-interaction](skills/state-art-llm-enabled-interaction/SKILL.md) | Data Processing | 6/10 | 258 |
| [state-transition-framework-reasoning](skills/state-transition-framework-reasoning/SKILL.md) | Reasoning & Chain-of-Thought | 5/10 | 215 |
| [stateless-yet-not-forgetful](skills/stateless-yet-not-forgetful/SKILL.md) | Security & Safety | 6/10 | 245 |
| [status-hierarchies](skills/status-hierarchies/SKILL.md) | Multi-Agent Systems | 5/10 | 229 |
| [steereval-framework-evaluating-steerability](skills/steereval-framework-evaluating-steerability/SKILL.md) | Evaluation & Benchmarking | 4/10 | 191 |
| [stepshield-not-whether-intervene](skills/stepshield-not-whether-intervene/SKILL.md) | Security & Safety | 6/10 | 282 |
| [stop-testing-attacks-start](skills/stop-testing-attacks-start/SKILL.md) | Security & Safety | 5/10 | 222 |
| [strong-reasoning-isnt-enough](skills/strong-reasoning-isnt-enough/SKILL.md) | Agentic Systems | 6/10 | 251 |
| [structured-context-engineering-file-native](skills/structured-context-engineering-file-native/SKILL.md) | Memory & Context | 7/10 | 302 |
| [supchain-bench-benchmarking-real-world-supply](skills/supchain-bench-benchmarking-real-world-supply/SKILL.md) | Agentic Systems | 5/10 | 198 |
| [supporting-software-engineering-tasks](skills/supporting-software-engineering-tasks/SKILL.md) | Code & Software Engineering | 5/10 | 169 |
| [svrepair-structured-visual-reasoning](skills/svrepair-structured-visual-reasoning/SKILL.md) | Multimodal | 7/10 | 193 |
| [swe-agi-benchmarking-specification-driven-software](skills/swe-agi-benchmarking-specification-driven-software/SKILL.md) | Code & Software Engineering | 7/10 | 189 |
| [swe-bench-mobile-agents-develop](skills/swe-bench-mobile-agents-develop/SKILL.md) | Code & Software Engineering | 7/10 | 247 |
| [swe-context-bench-benchmark](skills/swe-context-bench-benchmark/SKILL.md) | Memory & Context | 6/10 | 180 |
| [swe-manager-selecting-synthesizing-golden](skills/swe-manager-selecting-synthesizing-golden/SKILL.md) | Reasoning & Chain-of-Thought | 7/10 | 254 |
| [swe-master-unleashing-potential-software](skills/swe-master-unleashing-potential-software/SKILL.md) | Code & Software Engineering | 7/10 | 195 |
| [swe-pruner-self-adaptive-context-pruning](skills/swe-pruner-self-adaptive-context-pruning/SKILL.md) | Efficiency & Optimization | 7/10 | 186 |
| [swe-refactor-repository-level-benchmark-real-world](skills/swe-refactor-repository-level-benchmark-real-world/SKILL.md) | Code & Software Engineering | 7/10 | 151 |
| [swe-replay-test-time-scaling-software](skills/swe-replay-test-time-scaling-software/SKILL.md) | Efficiency & Optimization | 4/10 | 158 |
| [sycoeval-em-sycophancy-evaluation-simulated](skills/sycoeval-em-sycophancy-evaluation-simulated/SKILL.md) | Evaluation & Benchmarking | 5/10 | 241 |
| [symphony-synergistic-multi-agent-planning](skills/symphony-synergistic-multi-agent-planning/SKILL.md) | Multi-Agent Systems | 5/10 | 201 |
| [synthagent-multi-agent-framework-realistic](skills/synthagent-multi-agent-framework-realistic/SKILL.md) | Domain-Specific | 5/10 | 298 |
| [synthesizing-file-level-data-unit](skills/synthesizing-file-level-data-unit/SKILL.md) | Code & Software Engineering | 7/10 | 222 |
| [system-name-address-parsing](skills/system-name-address-parsing/SKILL.md) | Data Processing | 7/10 | 207 |
| [t2vtree-user-centered-visual-analytics](skills/t2vtree-user-centered-visual-analytics/SKILL.md) | Agentic Systems | 4/10 | 268 |
| [table-as-search-formulate-long-horizon-agentic](skills/table-as-search-formulate-long-horizon-agentic/SKILL.md) | Agentic Systems | 7/10 | 203 |
| [tam-eval-evaluating-llms-for](skills/tam-eval-evaluating-llms-for/SKILL.md) | Code & Software Engineering | 7/10 | 225 |
| [taming-scylla-understanding-multi-headed](skills/taming-scylla-understanding-multi-headed/SKILL.md) | Evaluation & Benchmarking | 5/10 | 165 |
| [teaching-evaluating-reason-about](skills/teaching-evaluating-reason-about/SKILL.md) | Domain-Specific | 4/10 | 202 |
| [test-vs-mutant-adversarial](skills/test-vs-mutant-adversarial/SKILL.md) | Code & Software Engineering | 8/10 | 221 |
| [testexplora-benchmarking-proactive-bug](skills/testexplora-benchmarking-proactive-bug/SKILL.md) | Evaluation & Benchmarking | 7/10 | 218 |
| [text-summarization-global-structure](skills/text-summarization-global-structure/SKILL.md) | NLP & Text | 5/10 | 165 |
| [textual-equilibrium-propagation-deep](skills/textual-equilibrium-propagation-deep/SKILL.md) | Prompt Engineering | 5/10 | 147 |
| [the-clef-2026-finmmeval-lab](skills/the-clef-2026-finmmeval-lab/SKILL.md) | Evaluation & Benchmarking | 4/10 | 246 |
| [the-compliance-paradox-semantic-instruction](skills/the-compliance-paradox-semantic-instruction/SKILL.md) | Security & Safety | 6/10 | 229 |
| [the-landscape-prompt-injection](skills/the-landscape-prompt-injection/SKILL.md) | Security & Safety | 7/10 | 244 |
| [the-necessity-unified-framework](skills/the-necessity-unified-framework/SKILL.md) | Evaluation & Benchmarking | 5/10 | 185 |
| [the-semantic-trap-fine-tuned](skills/the-semantic-trap-fine-tuned/SKILL.md) | Security & Safety | 7/10 | 179 |
| [the-shadow-self-intrinsic](skills/the-shadow-self-intrinsic/SKILL.md) | Security & Safety | 5/10 | 234 |
| [think-augmented-function-calling-improving](skills/think-augmented-function-calling-improving/SKILL.md) | Prompt Engineering | 7/10 | 262 |
| [thinking-makes-agents-introverted](skills/thinking-makes-agents-introverted/SKILL.md) | Prompt Engineering | 7/10 | 244 |
| [thinktank-me-multi-expert-framework-middle](skills/thinktank-me-multi-expert-framework-middle/SKILL.md) | Multi-Agent Systems | 4/10 | 207 |
| [timely-machine-awareness-time](skills/timely-machine-awareness-time/SKILL.md) | Efficiency & Optimization | 6/10 | 171 |
| [timemachine-bench-benchmark-evaluating-capabilitie](skills/timemachine-bench-benchmark-evaluating-capabilitie/SKILL.md) | Code & Software Engineering | 7/10 | 143 |
| [tokenomics-quantifying-where-tokens](skills/tokenomics-quantifying-where-tokens/SKILL.md) | Efficiency & Optimization | 5/10 | 227 |
| [toolself-unifying-task-execution](skills/toolself-unifying-task-execution/SKILL.md) | Agentic Systems | 6/10 | 217 |
| [toolweaver-weaving-collaborative-semantics](skills/toolweaver-weaving-collaborative-semantics/SKILL.md) | RAG & Retrieval | 4/10 | 181 |
| [toward-culturally-aligned-ontology-guided](skills/toward-culturally-aligned-ontology-guided/SKILL.md) | Knowledge Graphs | 3/10 | 190 |
| [toward-universal-transferable-jailbreak](skills/toward-universal-transferable-jailbreak/SKILL.md) | Security & Safety | 5/10 | 317 |
| [towards-adaptive-scalable-robust](skills/towards-adaptive-scalable-robust/SKILL.md) | Multi-Agent Systems | 5/10 | 205 |
| [towards-ai-evaluation-domain-specific](skills/towards-ai-evaluation-domain-specific/SKILL.md) | RAG & Retrieval | 7/10 | 260 |
| [towards-automated-kernel-generation](skills/towards-automated-kernel-generation/SKILL.md) | Efficiency & Optimization | 6/10 | 159 |
| [towards-autonomous-mathematics-research](skills/towards-autonomous-mathematics-research/SKILL.md) | Reasoning & Chain-of-Thought | 6/10 | 239 |
| [towards-declarative-agentic-layer](skills/towards-declarative-agentic-layer/SKILL.md) | Agentic Systems | 6/10 | 218 |
| [towards-green-ai-decoding](skills/towards-green-ai-decoding/SKILL.md) | Efficiency & Optimization | 5/10 | 229 |
| [towards-science-collective-ai](skills/towards-science-collective-ai/SKILL.md) | Multi-Agent Systems | 5/10 | 184 |
| [tracecoder-trace-driven-multi-agent-framework](skills/tracecoder-trace-driven-multi-agent-framework/SKILL.md) | Code & Software Engineering | 7/10 | 191 |
| [tracellm-leveraging-prompt-engineering](skills/tracellm-leveraging-prompt-engineering/SKILL.md) | Prompt Engineering | 6/10 | 179 |
| [tracemem-weaving-narrative-memory](skills/tracemem-weaving-narrative-memory/SKILL.md) | Memory & Context | 6/10 | 229 |
| [trapped-past-disentangling-fluid](skills/trapped-past-disentangling-fluid/SKILL.md) | Evaluation & Benchmarking | 4/10 | 174 |
| [treetensor-boost-ai-system](skills/treetensor-boost-ai-system/SKILL.md) | Data Processing | 5/10 | 237 |
| [tricky2-benchmark-evaluating-human-error](skills/tricky2-benchmark-evaluating-human-error/SKILL.md) | Code & Software Engineering | 6/10 | 207 |
| [triplay-rl-tri-role-self-play-reinforcement](skills/triplay-rl-tri-role-self-play-reinforcement/SKILL.md) | Security & Safety | 5/10 | 208 |
| [trust-design-skill-profiles](skills/trust-design-skill-profiles/SKILL.md) | Efficiency & Optimization | 5/10 | 215 |
| [ts-debate-multimodal-collaborative-debate](skills/ts-debate-multimodal-collaborative-debate/SKILL.md) | Multi-Agent Systems | 5/10 | 232 |
| [tsaqa-time-series-analysis](skills/tsaqa-time-series-analysis/SKILL.md) | Data Processing | 5/10 | 195 |
| [tsrbench-comprehensive-multi-task-multi-modal](skills/tsrbench-comprehensive-multi-task-multi-modal/SKILL.md) | Evaluation & Benchmarking | 4/10 | 206 |
| [tutorial-reasoning-ir-ir](skills/tutorial-reasoning-ir-ir/SKILL.md) | RAG & Retrieval | 5/10 | 249 |
| [uncertainty-and-fairness-awareness](skills/uncertainty-and-fairness-awareness/SKILL.md) | Evaluation & Benchmarking | 5/10 | 230 |
| [understanding-agent-scaling-llm-based](skills/understanding-agent-scaling-llm-based/SKILL.md) | Multi-Agent Systems | 6/10 | 204 |
| [understanding-dominant-themes-reviewing](skills/understanding-dominant-themes-reviewing/SKILL.md) | Evaluation & Benchmarking | 5/10 | 226 |
| [unicog-uncovering-cognitive-abilities](skills/unicog-uncovering-cognitive-abilities/SKILL.md) | Reasoning & Chain-of-Thought | 4/10 | 178 |
| [unicomp-unified-evaluation-compression](skills/unicomp-unified-evaluation-compression/SKILL.md) | Efficiency & Optimization | 4/10 | 177 |
| [unikie-bench-benchmarking-multimodal-key](skills/unikie-bench-benchmarking-multimodal-key/SKILL.md) | Multimodal | 5/10 | 292 |
| [unit-based-agent-semi-cascaded-full-duplex](skills/unit-based-agent-semi-cascaded-full-duplex/SKILL.md) | Domain-Specific | 5/10 | 247 |
| [unveiling-cognitive-compass-theory-of-mind-guided](skills/unveiling-cognitive-compass-theory-of-mind-guided/SKILL.md) | Reasoning & Chain-of-Thought | 4/10 | 197 |
| [usage-effects-requirements-ai-coding](skills/usage-effects-requirements-ai-coding/SKILL.md) | Prompt Engineering | 5/10 | 228 |
| [use-graph-it-needs](skills/use-graph-it-needs/SKILL.md) | RAG & Retrieval | 7/10 | 254 |
| [variability-aware-detection-repair-compilation-err](skills/variability-aware-detection-repair-compilation-err/SKILL.md) | Code & Software Engineering | 7/10 | 180 |
| [vectra-metric-dataset-visual](skills/vectra-metric-dataset-visual/SKILL.md) | Evaluation & Benchmarking | 5/10 | 305 |
| [verge-formal-refinement-guidance](skills/verge-formal-refinement-guidance/SKILL.md) | Reasoning & Chain-of-Thought | 5/10 | 185 |
| [veri-sure-contract-aware-multi-agent-framework](skills/veri-sure-contract-aware-multi-agent-framework/SKILL.md) | Domain-Specific | 5/10 | 186 |
| [vihermes-graph-grounded-multihop-question](skills/vihermes-graph-grounded-multihop-question/SKILL.md) | Knowledge Graphs | 6/10 | 264 |
| [villain-at-averimatec-verifying](skills/villain-at-averimatec-verifying/SKILL.md) | Multi-Agent Systems | 5/10 | 248 |
| [viola-video-in-context-learning](skills/viola-video-in-context-learning/SKILL.md) | RAG & Retrieval | 4/10 | 221 |
| [vision-deepresearch-incentivizing-deepresearch-cap](skills/vision-deepresearch-incentivizing-deepresearch-cap/SKILL.md) | Multimodal | 5/10 | 180 |
| [visiontrim-unified-vision-token](skills/visiontrim-unified-vision-token/SKILL.md) | Efficiency & Optimization | 4/10 | 212 |
| [vista-scene-aware-optimization-streaming](skills/vista-scene-aware-optimization-streaming/SKILL.md) | Multimodal | 4/10 | 261 |
| [vistira-closing-image-text-modality](skills/vistira-closing-image-text-modality/SKILL.md) | Multimodal | 6/10 | 189 |
| [visual-cognitive-demands-model-powered](skills/visual-cognitive-demands-model-powered/SKILL.md) | Domain-Specific | 4/10 | 290 |
| [vln-pilot-vision-language-as-autonomous](skills/vln-pilot-vision-language-as-autonomous/SKILL.md) | Agentic Systems | 5/10 | 235 |
| [vtc-r1-vision-text-compression-long-context](skills/vtc-r1-vision-text-compression-long-context/SKILL.md) | Efficiency & Optimization | 4/10 | 269 |
| [vulread-knowledge-graph-guided-software-vulnerabil](skills/vulread-knowledge-graph-guided-software-vulnerabil/SKILL.md) | Security & Safety | 7/10 | 212 |
| [wdscaling-parallel-tool-calling-deep](skills/wdscaling-parallel-tool-calling-deep/SKILL.md) | Reasoning & Chain-of-Thought | 7/10 | 161 |
| [what-should-cite-rag](skills/what-should-cite-rag/SKILL.md) | RAG & Retrieval | 5/10 | 193 |
| [when-agents-fail-act](skills/when-agents-fail-act/SKILL.md) | Agentic Systems | 6/10 | 194 |
| [when-agents-fail-comprehensive](skills/when-agents-fail-comprehensive/SKILL.md) | Agentic Systems | 8/10 | 240 |
| [when-agents-misremember-collectively](skills/when-agents-misremember-collectively/SKILL.md) | Multi-Agent Systems | 5/10 | 225 |
| [when-better-prompts-hurt](skills/when-better-prompts-hurt/SKILL.md) | Evaluation & Benchmarking | 7/10 | 196 |
| [when-get-significantly-worse](skills/when-get-significantly-worse/SKILL.md) | Evaluation & Benchmarking | 6/10 | 245 |
| [when-iterative-rag-beats](skills/when-iterative-rag-beats/SKILL.md) | RAG & Retrieval | 7/10 | 244 |
| [when-meets-fuzzy-topsis-personnel](skills/when-meets-fuzzy-topsis-personnel/SKILL.md) | Data Processing | 5/10 | 233 |
| [when-much-imagine-adaptive](skills/when-much-imagine-adaptive/SKILL.md) | Efficiency & Optimization | 7/10 | 228 |
| [when-should-search-more](skills/when-should-search-more/SKILL.md) | RAG & Retrieval | 7/10 | 300 |
| [where-ai-coding-agents](skills/where-ai-coding-agents/SKILL.md) | Code & Software Engineering | 7/10 | 193 |
| [whispers-wealth-red-teaming-googles](skills/whispers-wealth-red-teaming-googles/SKILL.md) | Security & Safety | 6/10 | 218 |
| [whitespaces-dont-lie-feature-driven](skills/whitespaces-dont-lie-feature-driven/SKILL.md) | Code & Software Engineering | 6/10 | 255 |
| [who-gets-which-message](skills/who-gets-which-message/SKILL.md) | Evaluation & Benchmarking | 5/10 | 221 |
| [why-ai-agents-systematically](skills/why-ai-agents-systematically/SKILL.md) | Multi-Agent Systems | 6/10 | 309 |
| [why-deep-research-agent](skills/why-deep-research-agent/SKILL.md) | Evaluation & Benchmarking | 5/10 | 199 |
| [why-reasoning-fails-plan](skills/why-reasoning-fails-plan/SKILL.md) | Reasoning & Chain-of-Thought | 7/10 | 218 |
| [wideseek-r1-exploring-width-scaling](skills/wideseek-r1-exploring-width-scaling/SKILL.md) | Multi-Agent Systems | 7/10 | 197 |
| [wiki-live-challenge-challenging](skills/wiki-live-challenge-challenging/SKILL.md) | Evaluation & Benchmarking | 5/10 | 264 |
| [will-it-survive-deciphering](skills/will-it-survive-deciphering/SKILL.md) | Code & Software Engineering | 5/10 | 173 |
| [world-workflows-benchmark-bringing](skills/world-workflows-benchmark-bringing/SKILL.md) | Domain-Specific | 6/10 | 185 |
| [xai-clip-roi-guided-perturbation-framework](skills/xai-clip-roi-guided-perturbation-framework/SKILL.md) | Explainability | 4/10 | 226 |
| [xlist-hate-checklist-based-framework-interpretable](skills/xlist-hate-checklist-based-framework-interpretable/SKILL.md) | Explainability | 6/10 | 229 |
| [yasa-scalable-multi-language-taint](skills/yasa-scalable-multi-language-taint/SKILL.md) | Security & Safety | 7/10 | 204 |
| [yoloe-26-integrating-yolo26-yoloe](skills/yoloe-26-integrating-yolo26-yoloe/SKILL.md) | Domain-Specific | 5/10 | 232 |
| [yunque-deepresearch-technical-report](skills/yunque-deepresearch-technical-report/SKILL.md) | Agentic Systems | 6/10 | 191 |
| [zero-shot-product-attribute-labeling](skills/zero-shot-product-attribute-labeling/SKILL.md) | Multimodal | 5/10 | 268 |
