# Skill Index

**1033 skills** organized by category. Each skill encodes a research technique from arXiv as an actionable Claude Code workflow.

Most skills span multiple categories. They appear under each relevant heading.

## Categories

| Category | Skills | Avg Lines | Description |
|----------|--------|-----------|-------------|
| [Data Processing](#data-processing) | 431 | 214 | ETL, parsing, extraction, annotation, pipelines |
| [Agentic Systems](#agentic-systems) | 346 | 211 | Autonomous agents, tool use, planning, task decomposition |
| [Evaluation & Benchmarking](#evaluation-benchmarking) | 263 | 214 | Benchmarks, metrics, scoring, model assessment |
| [Efficiency & Optimization](#efficiency-optimization) | 260 | 206 | Quantization, pruning, compression, acceleration |
| [Reasoning & Chain-of-Thought](#reasoning-chain-of-thought) | 259 | 211 | Logical reasoning, CoT, step-by-step inference, math |
| [RAG & Retrieval](#rag-retrieval) | 207 | 206 | Retrieval-augmented generation, search, reranking, document retrieval |
| [Fine-tuning & Training](#fine-tuning-training) | 191 | 213 | RLHF, GRPO, distillation, curriculum learning, reward models |
| [Multimodal](#multimodal) | 175 | 228 | Vision-language, audio, video, speech, cross-modal |
| [Multi-Agent Systems](#multi-agent-systems) | 124 | 209 | Agent collaboration, swarms, orchestration, debate |
| [Prompt Engineering](#prompt-engineering) | 123 | 210 | In-context learning, few-shot, instruction tuning |
| [Security & Safety](#security-safety) | 116 | 223 | Jailbreaks, adversarial attacks, guardrails, prompt injection, red teaming |
| [Memory & Context](#memory-context) | 109 | 213 | Long-context handling, KV cache, attention, context compression |
| [Code & Software Engineering](#code-software-engineering) | 108 | 198 | Code generation, bug detection, testing, refactoring |
| [Other](#other) | 90 | 217 | Novel techniques that don't fit standard categories |
| [NLP & Text](#nlp-text) | 80 | 211 | Classification, summarization, translation, NER, QA |
| [Explainability](#explainability) | 78 | 211 | Interpretability, attribution, causal analysis |
| [Domain-Specific](#domain-specific) | 47 | 222 | Medical, legal, financial, clinical applications |
| [Knowledge Graphs](#knowledge-graphs) | 38 | 210 | Graph-based knowledge, ontologies, entity relations |

---

## Data Processing

**431 skills**

| Skill | Description | Lines |
|-------|-------------|-------|
| [3-secbench-large-scale-evaluation-suite-security](skills/3-secbench-large-scale-evaluation-suite-security/SKILL.md) | Evaluate and harden LLM-based autonomous agents against adversarial attacks using the α³-SecBench layered security frame... | 182 |
| [3d-space-as-scratchpad-editable](skills/3d-space-as-scratchpad-editable/SKILL.md) | Build agentic pipelines that use 3D scene layout as an intermediate reasoning workspace for controllable, spatially-accu... | 172 |
| [a2-llm-end-to-end-conversational-audio-avatar](skills/a2-llm-end-to-end-conversational-audio-avatar/SKILL.md) | Build end-to-end conversational audio avatar systems that jointly generate speech and expressive 3D facial motion from a... | 244 |
| [a2rag-adaptive-agentic-graph](skills/a2rag-adaptive-agentic-graph/SKILL.md) | Build adaptive, cost-aware Graph-RAG pipelines that route queries through escalating retrieval stages (local -> bridge -... | 229 |
| [accelerating-social-science-research](skills/accelerating-social-science-research/SKILL.md) | Implement the EXPERIGEN agentic framework for automated hypothesis generation and empirical validation on datasets. Uses... | 177 |
| [acegrpo-adaptive-curriculum-group](skills/acegrpo-adaptive-curriculum-group/SKILL.md) | Adaptive curriculum-driven iterative optimization for autonomous ML engineering tasks. Uses Evolving Data Buffers and Le... | 220 |
| [adareasoner-dynamic-tool-orchestration](skills/adareasoner-dynamic-tool-orchestration/SKILL.md) | Adaptive multi-step tool orchestration for complex reasoning tasks. Dynamically selects, sequences, and composes tools b... | 168 |
| [addressing-explainability-generative-ai](skills/addressing-explainability-generative-ai/SKILL.md) | Explain generative AI outputs using the gSMILE perturbation-based attribution framework. Builds local surrogate models f... | 222 |
| [agent-primitives-reusable-latent](skills/agent-primitives-reusable-latent/SKILL.md) | Design and orchestrate multi-agent systems using reusable Agent Primitives (Review, Voting/Selection, Planning/Execution... | 252 |
| [agent2agent-threats-safety-critical-assistants](skills/agent2agent-threats-safety-critical-assistants/SKILL.md) | Threat model multi-agent LLM systems using the AgentHeLLM framework -- formally separating asset identification from att... | 207 |
| [agentark-distilling-multi-agent-intelligence](skills/agentark-distilling-multi-agent-intelligence/SKILL.md) | Distill multi-agent debate reasoning into a single LLM's behavior. Apply AgentArk's three-tier distillation strategy to ... | 199 |
| [agentdrive-open-benchmark-dataset](skills/agentdrive-open-benchmark-dataset/SKILL.md) | Generate structured autonomous driving scenarios and MCQ benchmarks using AgentDrive's factorized 7-axis prompt-to-JSON ... | 284 |
| [agentic-reinforcement-learning-empowers](skills/agentic-reinforcement-learning-empowers/SKILL.md) | Build tool-augmented agent systems that decouple domain reasoning from knowledge storage, following the ChemCRAFT patter... | 242 |
| [agenticpay-multi-agent-negotiation-system](skills/agenticpay-multi-agent-negotiation-system/SKILL.md) | Build multi-agent LLM negotiation systems where buyer and seller agents reach deals through natural language. Use when a... | 226 |
| [agenticsimlaw-juvenile-courtroom-multi-agent](skills/agenticsimlaw-juvenile-courtroom-multi-agent/SKILL.md) | Structured multi-agent courtroom debate for explainable high-stakes tabular decisions. Use when: 'set up a multi-agent d... | 181 |
| [agentskiller-scaling-generalist-agent](skills/agentskiller-scaling-generalist-agent/SKILL.md) | Synthesize multi-turn agent interaction data across semantically linked domains using DAG-based pipelines, domain ontolo... | 178 |
| [agentsm-semantic-memory-agentic](skills/agentsm-semantic-memory-agentic/SKILL.md) | Agentic Text-to-SQL with semantic memory that captures and reuses structured execution traces. Use when: 'write SQL for ... | 213 |
| [agentxray-white-boxing-agentic-systems](skills/agentxray-white-boxing-agentic-systems/SKILL.md) | Reverse-engineer black-box agentic systems into editable, interpretable workflows using search-based reconstruction. Use... | 168 |
| [ai-my-values-user](skills/ai-my-values-user/SKILL.md) | Build value-aligned conversational agents using the VAPT (Value-Alignment Perception Toolkit) framework from CHI '26. Ex... | 236 |
| [aiano-enhancing-information-retrieval](skills/aiano-enhancing-information-retrieval/SKILL.md) | Build AI-augmented annotation pipelines for creating high-quality information retrieval and QA datasets. Combines LLM-ge... | 161 |
| [aicd-bench-challenging-benchmark](skills/aicd-bench-challenging-benchmark/SKILL.md) | Detect whether source code was written by a human or generated by an AI model, attribute code to specific model families... | 211 |
| [alienlm-alienization-api-boundary-privacy](skills/alienlm-alienization-api-boundary-privacy/SKILL.md) | Implement AlienLM-style API-boundary privacy layers that protect sensitive text sent to black-box LLM APIs using vocabul... | 202 |
| [ama-adaptive-memory-multi-agent](skills/ama-adaptive-memory-multi-agent/SKILL.md) | Build adaptive memory systems using coordinated multi-agent collaboration with hierarchical storage and consistency main... | 224 |
| [an-cost-efficient-agentic-framework](skills/an-cost-efficient-agentic-framework/SKILL.md) | Audit Ethereum smart contracts for business logic vulnerabilities using Heimdallr's four-phase agentic pipeline: functio... | 202 |
| [analyticsgpt-workflow-scientometric-question](skills/analyticsgpt-workflow-scientometric-question/SKILL.md) | Build sequential LLM pipelines for scientometric question answering over academic databases. Decomposes meta-scientific ... | 306 |
| [annotation-free-spacecraft-detection](skills/annotation-free-spacecraft-detection/SKILL.md) | > | 229 |
| [anonymization-enhanced-privacy-protection-mobile-g](skills/anonymization-enhanced-privacy-protection-mobile-g/SKILL.md) | Implement available-but-invisible privacy protection for mobile GUI agents using PII-aware anonymization with determinis... | 178 |
| [aorchestra-automating-sub-agent-creation](skills/aorchestra-automating-sub-agent-creation/SKILL.md) | Dynamically create specialized sub-agents for complex multi-step tasks using the AOrchestra pattern: decompose goals, th... | 203 |
| [are-open-weight-ready-social](skills/are-open-weight-ready-social/SKILL.md) | Build LLM-based content moderation pipelines using zero-shot classification with open-weight models. Implements the stru... | 264 |
| [asa-training-free-representation-engineering](skills/asa-training-free-representation-engineering/SKILL.md) | Implement Activation Steering Adapter (ASA) for training-free tool-calling improvement in LLM agents. Use when: 'fix laz... | 170 |
| [assessment-generative-named-entity](skills/assessment-generative-named-entity/SKILL.md) | Build generative NER systems using LLMs with optimal output formats and prompt engineering. Use when: 'extract entities ... | 245 |
| [attn-gs-attention-guided-context-compression](skills/attn-gs-attention-guided-context-compression/SKILL.md) | Compress long user contexts (profiles, histories, documents) into concise, high-quality summaries using attention-guided... | 158 |
| [audiorouter-data-audio-understanding](skills/audiorouter-data-audio-understanding/SKILL.md) | Build audio understanding systems that route between internal LLM reasoning and external audio tools using a lightweight... | 249 |
| [audit-after-segmentation-reference-free](skills/audit-after-segmentation-reference-free/SKILL.md) | Build reference-free mask quality assessment pipelines for multimodal segmentation systems. Implements the MQ-Auditor pa... | 251 |
| [automated-customization-enterprise-code](skills/automated-customization-enterprise-code/SKILL.md) | Customize LLMs for enterprise code repositories using semantic scopes -- automatically partition codebases into meaningf... | 153 |
| [automated-multiple-mini-interview](skills/automated-multiple-mini-interview/SKILL.md) | Multi-agent framework for scoring subjective, open-ended responses (interviews, essays, reflections) using transcript re... | 189 |
| [automated-rubrics-reliable-evaluation](skills/automated-rubrics-reliable-evaluation/SKILL.md) | Generate fine-grained evaluation rubrics for medical dialogue systems using a retrieval-augmented multi-agent pipeline. ... | 180 |
| [automating-computational-reproducibility-social](skills/automating-computational-reproducibility-social/SKILL.md) | Diagnose and repair failing computational research code to restore reproducibility. Uses an agent-based iterative workfl... | 189 |
| [autonomous-chain-of-thought-distillation-graph-bas](skills/autonomous-chain-of-thought-distillation-graph-bas/SKILL.md) | Implement FraudCoT-style graph-aware chain-of-thought distillation for fraud detection on text-attributed graphs. Combin... | 199 |
| [autonomous-data-processing-meta-agents](skills/autonomous-data-processing-meta-agents/SKILL.md) | Build self-managing data processing pipelines using hierarchical meta-agent orchestration. Decomposes complex data tasks... | 216 |
| [avere-improving-audiovisual-emotion](skills/avere-improving-audiovisual-emotion/SKILL.md) | Build emotion-aware multimodal AI systems that resist spurious cue-emotion associations and hallucinated audiovisual evi... | 177 |
| [bass-benchmarking-audio-lms](skills/bass-benchmarking-audio-lms/SKILL.md) | Build evaluation benchmarks for audio language models using the BASS methodology — structured task taxonomies across str... | 260 |
| [bayesflow-probability-inference-framework](skills/bayesflow-probability-inference-framework/SKILL.md) | Generate high-quality multi-step LLM workflows using Bayesian inference with parallel look-ahead rollouts and importance... | 204 |
| [better-as-generators-than](skills/better-as-generators-than/SKILL.md) | Generate synthetic labeled datasets with LLMs to train smaller, cheaper classifiers -- especially for low-resource langu... | 178 |
| [better-generalizing-unseen-concepts](skills/better-generalizing-unseen-concepts/SKILL.md) | Build biomedical concept recognition systems that generalize to unseen ontology concepts using hierarchical indexing and... | 143 |
| [beyond-accuracy-cognitive-load](skills/beyond-accuracy-cognitive-load/SKILL.md) | Analyze and reduce cognitive load in tool-use agent workflows using the Cognitive Load Framework from AAAI 2026. Diagnos... | 218 |
| [beyond-alignment-expanding-reasoning](skills/beyond-alignment-expanding-reasoning/SKILL.md) | Apply Manifold-Reshaping Policy Optimization (MRPO) to expand LLM reasoning capacity beyond alignment. Implements spectr... | 250 |
| [beyond-needles-illusion-decoupled](skills/beyond-needles-illusion-decoupled/SKILL.md) | Decouple evidence access from evidence use when evaluating or building long-context and RAG systems under semantic inter... | 191 |
| [beyond-speedup-utilizing](skills/beyond-speedup-utilizing/SKILL.md) | Reuse LLM KV caches as free embeddings for confidence scoring and adaptive fast/slow reasoning. Use when: 'extract embed... | 247 |
| [beyond-translation-cross-cultural-meme](skills/beyond-translation-cross-cultural-meme/SKILL.md) | Cross-cultural meme transcreation using a three-stage hybrid pipeline (cultural analysis, visual generation, assembly) t... | 170 |
| [biases-blind-spot-detecting](skills/biases-blind-spot-detecting/SKILL.md) | Automated black-box pipeline for detecting unverbalized biases in LLM decision-making. Discovers biases that models exhi... | 194 |
| [biasscope-automated-detection-bias](skills/biasscope-automated-detection-bias/SKILL.md) | Automatically discover and test for hidden biases in LLM-as-a-Judge evaluation pipelines using the BiasScope framework. ... | 206 |
| [birdturk-adaptation-bird-text-to-sql](skills/birdturk-adaptation-bird-text-to-sql/SKILL.md) | Adapt Text-to-SQL systems and benchmarks for non-English, morphologically rich languages using controlled translation pi... | 242 |
| [blind-gods-broken-screens](skills/blind-gods-broken-screens/SKILL.md) | Architect secure, intent-centric agent systems using the Aura pattern: Hub-and-Spoke agent topology, cryptographic ident... | 219 |
| [breaking-static-graph-context-aware](skills/breaking-static-graph-context-aware/SKILL.md) | Build query-adaptive knowledge graph retrieval systems using CatRAG's context-aware traversal. Transforms static KG-base... | 170 |
| [bridging-academia-industry-comprehensive](skills/bridging-academia-industry-comprehensive/SKILL.md) | Attributed graph clustering pipelines using PyAGC's Encode-Cluster-Optimize framework. Triggers: 'cluster nodes in a gra... | 264 |
| [bridging-arithmetic-gap-cognitive](skills/bridging-arithmetic-gap-cognitive/SKILL.md) | Iterative Dual-Phase Financial-PoT: decouple semantic reasoning from arithmetic computation to eliminate calculation err... | 222 |
| [bridging-lexical-ambiguity-vision](skills/bridging-lexical-ambiguity-vision/SKILL.md) | Build Visual Word Sense Disambiguation (VWSD) systems that resolve lexical ambiguity using CLIP, diffusion models, and L... | 188 |
| [bridging-modality-gap-roadside](skills/bridging-modality-gap-roadside/SKILL.md) | Build training-free pipelines that convert sparse 3D LiDAR point clouds into depth-encoded 2D images for classification ... | 211 |
| [c3box-clip-based-class-incremental-learning](skills/c3box-clip-based-class-incremental-learning/SKILL.md) | Set up, configure, and run CLIP-based class-incremental learning experiments using the C3Box toolbox. Supports 17 CIL al... | 215 |
| [cam-causality-based-analysis-framework](skills/cam-causality-based-analysis-framework/SKILL.md) | Analyze and optimize multi-agent code generation pipelines using causality-based importance ranking of intermediate feat... | 153 |
| [can-clean-up-mess](skills/can-clean-up-mess/SKILL.md) | LLM-driven data preparation pipeline for cleaning, integrating, and enriching messy datasets. Use when the user says 'cl... | 164 |
| [can-implement-agent-based-odd-based](skills/can-implement-agent-based-odd-based/SKILL.md) | Translate ODD protocol specifications into validated, executable agent-based model (ABM) code in Python. Use when the us... | 208 |
| [can-post-training-transform-causal](skills/can-post-training-transform-causal/SKILL.md) | Perform rigorous causal inference tasks using structured reasoning pipelines inspired by CauGym. Estimate treatment effe... | 179 |
| [can-reasoning-be-trusted](skills/can-reasoning-be-trusted/SKILL.md) | Validate and score LLM-generated statistical reasoning using a three-axis rubric (Correctness 40%, Explanation 35%, Reas... | 203 |
| [causal-perspective-enhancing-jailbreak-attack](skills/causal-perspective-enhancing-jailbreak-attack/SKILL.md) | Apply causal analysis to LLM safety: identify direct causal drivers of jailbreaks using prompt feature decomposition, bu... | 175 |
| [causaltad-injecting-causal-knowledge](skills/causaltad-injecting-causal-knowledge/SKILL.md) | Detect anomalies in tabular data by injecting causal column relationships into LLM-based detection pipelines. Reorders a... | 267 |
| [chain-simulation-dual-mode-reasoning](skills/chain-simulation-dual-mode-reasoning/SKILL.md) | Dual-mode reasoning framework that dynamically routes problems to specialized strategies: computational flow for math, s... | 175 |
| [chunking-retrieval-re-ranking-empirical-evaluation](skills/chunking-retrieval-re-ranking-empirical-evaluation/SKILL.md) | Build and optimize two-stage RAG pipelines with bi-encoder retrieval, cross-encoder re-ranking, and empirically-validate... | 230 |
| [cimrag-cim-aware-domain-adaptive-noise-resilient](skills/cimrag-cim-aware-domain-adaptive-noise-resilient/SKILL.md) | Build noise-resilient RAG retrieval pipelines for edge/resource-constrained deployments. Implements TONEL (Task-Oriented... | 227 |
| [closing-reasoning-gaps-clinical](skills/closing-reasoning-gaps-clinical/SKILL.md) | Build systems that detect and fix reasoning gaps in LLM agents by comparing their chain-of-thought against reference rea... | 196 |
| [cognitive-platform-engineering-autonomous](skills/cognitive-platform-engineering-autonomous/SKILL.md) | Build autonomous cloud operations using a four-plane cognitive architecture (Sensing, Reasoning, Orchestration, Experien... | 247 |
| [commcp-multi-agent-coordination-llm-based](skills/commcp-multi-agent-coordination-llm-based/SKILL.md) | Build decentralized multi-agent coordination systems using LLM-based communication calibrated with conformal prediction.... | 225 |
| [compar-ia-french-governments](skills/compar-ia-french-governments/SKILL.md) | Build multilingual LLM evaluation arenas and preference data collection pipelines modeled on France's compar:IA platform... | 310 |
| [completing-missing-annotation-multi-agent](skills/completing-missing-annotation-multi-agent/SKILL.md) | Multi-agent debate framework for relevance assessment and annotation completion. Uses opposing-stance LLM agents with it... | 251 |
| [comprehensive-comparison-rag-methods](skills/comprehensive-comparison-rag-methods/SKILL.md) | Select and configure the right RAG strategy for conversational QA systems based on dataset characteristics. Use when: 'b... | 167 |
| [computational-approach-visual-metonymy](skills/computational-approach-visual-metonymy/SKILL.md) | Generate and evaluate visual metonymy -- indirect visual representations that evoke concepts through associated cues rat... | 179 |
| [confounding-robust-continuous-control](skills/confounding-robust-continuous-control/SKILL.md) | Implement confounding-robust reward shaping for continuous control RL using causal Bellman upper bounds and PBRS. Use wh... | 205 |
| [consistency-meets-verification-enhancing](skills/consistency-meets-verification-enhancing/SKILL.md) | Generate high-reliability test suites without ground-truth implementations using the ConVerTest pipeline: Self-Consisten... | 178 |
| [constrained-process-maps-multi-agent](skills/constrained-process-maps-multi-agent/SKILL.md) | Build multi-agent workflows structured as constrained DAG process maps with Monte Carlo uncertainty estimation. Each age... | 244 |
| [constructing-multi-label-hierarchical-classificati](skills/constructing-multi-label-hierarchical-classificati/SKILL.md) | Build multi-label hierarchical classifiers for MITRE ATT&CK text tagging using stage-wise classical ML (SGD-SVM + TF-IDF... | 226 |
| [cord-bridging-audio-text-reasoning](skills/cord-bridging-audio-text-reasoning/SKILL.md) | Implement CORD (Cross-modal On-policy Distillation) to bridge modality gaps in multimodal AI systems. Applies weighted s... | 238 |
| [core-comprehensive-ontological-relation](skills/core-comprehensive-ontological-relation/SKILL.md) | Detect and prevent semantic collapse in LLM outputs — where models fabricate spurious relationships between unrelated co... | 216 |
| [core-ubiquitous-6g-intelligence](skills/core-ubiquitous-6g-intelligence/SKILL.md) | Design and implement multi-LLM agent orchestration systems over hierarchical compute tiers using the CORE framework patt... | 195 |
| [corpusqa-10-million-token](skills/corpusqa-10-million-token/SKILL.md) | Corpus-level QA over massive document collections using memory-augmented agentic processing. Synthesize answers that req... | 179 |
| [cost-efficient-rag-entity-matching](skills/cost-efficient-rag-entity-matching/SKILL.md) | Build cost-efficient RAG pipelines for entity matching and deduplication using blocking-based batch retrieval and genera... | 187 |
| [craft-calibrated-reasoning-answer-faithful](skills/craft-calibrated-reasoning-answer-faithful/SKILL.md) | Apply CRAFT (Calibrated Reasoning with Answer-Faithful Traces) for multi-hop question answering with verified reasoning ... | 192 |
| [creditaudit-2textnd-dimension-evaluation](skills/creditaudit-2textnd-dimension-evaluation/SKILL.md) | Evaluate and select LLMs using CreditAudit's 2D framework: mean ability plus stability risk (fluctuation) across system ... | 211 |
| [cross-lingual-stability-judges-under](skills/cross-lingual-stability-judges-under/SKILL.md) | Detect and fix cross-lingual evaluation instabilities in LLM-as-a-judge pipelines. Use when: 'audit my multilingual eval... | 199 |
| [cure-curriculum-guided-multi-task-training](skills/cure-curriculum-guided-multi-task-training/SKILL.md) | Implement error-aware curriculum learning for multi-task training pipelines, especially medical/vision-language models. ... | 234 |
| [d-orca-dialogue-centric-optimization-robust](skills/d-orca-dialogue-centric-optimization-robust/SKILL.md) | Build dialogue-centric audio-visual captioning pipelines that identify who spoke what and when in multi-party video conv... | 228 |
| [d2quant-accurate-low-bit-post-training-weight](skills/d2quant-accurate-low-bit-post-training-weight/SKILL.md) | Apply the D2Quant post-training weight quantization framework to compress LLMs to sub-4-bit precision (2-bit, 3-bit) wit... | 229 |
| [darwin-dynamic-agentically-rewriting](skills/darwin-dynamic-agentically-rewriting/SKILL.md) | Evolutionary multi-agent code optimization using genetic algorithms. Agents mutate each other's training/configuration c... | 167 |
| [data-centric-interpretability-llm-based-multi-agen](skills/data-centric-interpretability-llm-based-multi-agen/SKILL.md) | Analyze LLM agent behavior across training runs using Sparse Autoencoder (SAE) features and LLM-summarizer pipelines. Gr... | 189 |
| [datachef-cooking-up-optimal](skills/datachef-cooking-up-optimal/SKILL.md) | Automate data recipe generation for LLM fine-tuning and adaptation. Generates executable data processing pipelines (filt... | 207 |
| [datacross-unified-benchmark-agent](skills/datacross-unified-benchmark-agent/SKILL.md) | Cross-modal data analysis agent that unifies structured sources (SQL, CSV, JSON) with unstructured visual documents (sca... | 203 |
| [decoupled-reasoning-implicit-fact](skills/decoupled-reasoning-implicit-fact/SKILL.md) | Build dual-model pipelines that decouple knowledge extraction from reasoning over long documents. Compress document chun... | 164 |
| [decoupling-skeleton-flesh-multimodal](skills/decoupling-skeleton-flesh-multimodal/SKILL.md) | Disentangled structure-content reasoning for table images and structured data. Separates table skeleton (layout/structur... | 180 |
| [deep-search-hierarchical-meta-cognitive](skills/deep-search-hierarchical-meta-cognitive/SKILL.md) | Implement hierarchical meta-cognitive monitoring for deep search agents. Embeds a two-tier self-monitoring system (fast ... | 188 |
| [deepasmr-llm-based-zero-shot-asmr](skills/deepasmr-llm-based-zero-shot-asmr/SKILL.md) | Build zero-shot ASMR speech generation systems using a two-stage LLM + flow-matching pipeline that separates speaking st... | 227 |
| [deepera-deep-evidence-reranking](skills/deepera-deep-evidence-reranking/SKILL.md) | Rerank retrieved passages for RAG pipelines using step-by-step logical reasoning to filter out semantically similar but ... | 223 |
| [deepimagesearch-benchmarking-multimodal-agents](skills/deepimagesearch-benchmarking-multimodal-agents/SKILL.md) | Build agentic image retrieval systems that perform multi-step contextual reasoning over visual histories instead of isol... | 198 |
| [devops-gym-benchmarking-ai-agents](skills/devops-gym-benchmarking-ai-agents/SKILL.md) | Apply the DevOps-Gym methodology to systematically tackle full-cycle DevOps tasks: build/configuration repair, runtime m... | 170 |
| [diffa-2-practical-diffusion-general](skills/diffa-2-practical-diffusion-general/SKILL.md) | Design and implement diffusion-based large audio language models (LALMs) using the DIFFA-2 architecture — dual-adapter p... | 237 |
| [diffusion-pretrained-dense-contextual-embeddings](skills/diffusion-pretrained-dense-contextual-embeddings/SKILL.md) | Build production retrieval systems using pplx-embed, diffusion-pretrained dense and contextualized embedding models with... | 183 |
| [discovering-high-level-patterns](skills/discovering-high-level-patterns/SKILL.md) | Extract high-level semantic patterns from fine-grained simulation or event logs using LM-guided program synthesis. Trans... | 190 |
| [distilling-reasoning-graph-concept](skills/distilling-reasoning-graph-concept/SKILL.md) | Distill LLM reasoning into a DAG of modular concept predictors for efficient, interpretable classification. Use when ask... | 170 |
| [dllm-agent-see-farther](skills/dllm-agent-see-farther/SKILL.md) | Design and implement multi-agent workflows using the DeepDiver hierarchical orchestration pattern with diffusion-inspire... | 163 |
| [dllm-searcher-adapting-diffusion-large](skills/dllm-searcher-adapting-diffusion-large/SKILL.md) | Implement the P-ReAct parallel reasoning-and-acting agent paradigm from DLLM-Searcher, which overlaps tool execution wit... | 256 |
| [do-truly-benefit-longer](skills/do-truly-benefit-longer/SKILL.md) | Optimize LLM context length for post-editing and refinement pipelines. Applies research showing that naively adding docu... | 252 |
| [do-vlms-have-moral](skills/do-vlms-have-moral/SKILL.md) | Audit and harden the moral robustness of Vision-Language Model (VLM) pipelines against adversarial perturbations that fl... | 211 |
| [doc2spec-synthesizing-formal-programming](skills/doc2spec-synthesizing-formal-programming/SKILL.md) | Synthesize formal programming specifications from natural-language API docs using grammar induction. Extracts rules from... | 177 |
| [domain-adaptation-synthetic-data-fine-tuning-germa](skills/domain-adaptation-synthetic-data-fine-tuning-germa/SKILL.md) | Generate difficulty-graded synthetic QA datasets from authoritative domain documents (laws, regulations, standards) and ... | 193 |
| [dr-mas-stable-reinforcement-learning](skills/dr-mas-stable-reinforcement-learning/SKILL.md) | Design and implement stable reinforcement learning pipelines for multi-agent LLM systems using agent-wise advantage norm... | 203 |
| [draincode-stealthy-energy-consumption](skills/draincode-stealthy-energy-consumption/SKILL.md) | Evaluate and defend RAG-based code generation systems against energy-drain attacks that poison retrieval contexts to inf... | 221 |
| [drugr-optimizing-molecular-drugs](skills/drugr-optimizing-molecular-drugs/SKILL.md) | Optimize molecular drug candidates using LLM-based explicit pharmacological reasoning over SMILES structures. Applies th... | 187 |
| [duogen-general-purpose-interleaved](skills/duogen-general-purpose-interleaved/SKILL.md) | Design and implement interleaved multimodal generation pipelines that alternate between text and image generation using ... | 209 |
| [dynaweb-model-based-reinforcement-learning](skills/dynaweb-model-based-reinforcement-learning/SKILL.md) | Build model-based RL training pipelines for web agents using learned world models (environment simulators) that predict ... | 193 |
| [dziribot-rag-intelligent-conversational](skills/dziribot-rag-intelligent-conversational/SKILL.md) | Build dialect-aware RAG conversational agents that handle non-standard orthography, code-switching, and multi-script inp... | 236 |
| [ecg-r1-protocol-guided-modality-agnostic-mllm](skills/ecg-r1-protocol-guided-modality-agnostic-mllm/SKILL.md) | Build protocol-guided medical AI interpretation pipelines with structured diagnostic reasoning, modality-robust architec... | 260 |
| [echoes-loop-diagnosing-risks](skills/echoes-loop-diagnosing-risks/SKILL.md) | Diagnose and mitigate feedback-loop risks (bias amplification, hallucination propagation, exposure polarization) in LLM-... | 395 |
| [edge-optimized-vision-language-underground-infrast](skills/edge-optimized-vision-language-underground-infrast/SKILL.md) | Build edge-deployable two-stage pipelines that combine lightweight segmentation with quantized Vision-Language Models fo... | 483 |
| [eft-cot-multi-agent-chain-of-thought-framework](skills/eft-cot-multi-agent-chain-of-thought-framework/SKILL.md) | Build multi-agent emotion-focused therapy (EFT) reasoning pipelines for empathetic mental health Q&A systems. Uses a bot... | 300 |
| [emoara-emotion-preserving-english-speech](skills/emoara-emotion-preserving-english-speech/SKILL.md) | Build emotion-preserving cross-lingual speech pipelines that detect emotion from audio, transcribe, translate, and synth... | 217 |
| [emotion-llamav2-mmeverse-framework-benchmark](skills/emotion-llamav2-mmeverse-framework-benchmark/SKILL.md) | Build multimodal emotion understanding systems using the Emotion-LLaMAv2 architecture and MMEVerse benchmark methodology... | 232 |
| [emotionthinker-prosody-aware-reinforcement-learnin](skills/emotionthinker-prosody-aware-reinforcement-learnin/SKILL.md) | Build prosody-aware speech emotion reasoning pipelines using Chain-of-Thought RL. Implements EmotionThinker's GRPO-PTR t... | 292 |
| [epistemic-context-learning-building](skills/epistemic-context-learning-building/SKILL.md) | Build trust-aware multi-agent systems using Epistemic Context Learning (ECL). Constructs peer reliability profiles from ... | 210 |
| [es-memeval-benchmarking-conversational-agents](skills/es-memeval-benchmarking-conversational-agents/SKILL.md) | Build and evaluate long-term memory systems for conversational agents using the ES-MemEval five-capability framework (in... | 226 |
| [evaluating-kubernetes-performance-genai](skills/evaluating-kubernetes-performance-genai/SKILL.md) | Design and optimize Kubernetes-native GenAI inference platforms using Kueue job queuing, Dynamic Accelerator Slicer (DAS... | 217 |
| [evaluating-social-bias-rag](skills/evaluating-social-bias-rag/SKILL.md) | Evaluate and mitigate social bias in RAG pipelines. Use when: 'audit my RAG system for bias', 'check if retrieval introd... | 212 |
| [evaluating-they-not-know](skills/evaluating-they-not-know/SKILL.md) | Build statistically efficient LLM evaluation pipelines that combine direct accuracy with pairwise comparison signals as ... | 185 |
| [evaluation-entity-matching-recommender](skills/evaluation-entity-matching-recommender/SKILL.md) | Build and evaluate cross-dataset entity matching pipelines for recommender systems. Implements the Reddit-Amazon-EM meth... | 182 |
| [evaluation-legal-applications-challenges](skills/evaluation-legal-applications-challenges/SKILL.md) | Build evaluation pipelines for LLMs in legal tasks using a three-dimensional framework: outcome correctness, reasoning r... | 171 |
| [evaluation-oncotimia-system-supporting](skills/evaluation-oncotimia-system-supporting/SKILL.md) | Build RAG pipelines that transform unstructured clinical or domain-specific documents into structured form records using... | 213 |
| [event-vstream-event-driven-real-time-understanding](skills/event-vstream-event-driven-real-time-understanding/SKILL.md) | Build event-driven video stream processing pipelines that detect meaningful state transitions instead of processing ever... | 254 |
| [eventcast-hybrid-demand-forecasting](skills/eventcast-hybrid-demand-forecasting/SKILL.md) | Build hybrid demand forecasting systems that fuse LLM-extracted event knowledge with time-series models using a dual-tow... | 228 |
| [evermembench-benchmarking-long-term-interactive](skills/evermembench-benchmarking-long-term-interactive/SKILL.md) | Build and evaluate long-term conversational memory systems for multi-party, multi-topic dialogues. Implements the EverMe... | 182 |
| [evolving-tool-user-creator](skills/evolving-tool-user-creator/SKILL.md) | Transform Claude from a static tool user into a dynamic tool creator using the UCT (User-to-Creator Transformation) fram... | 181 |
| [ex-omni-enabling-3d-facial](skills/ex-omni-enabling-3d-facial/SKILL.md) | Build pipelines that generate synchronized 3D facial animation alongside speech from omni-modal LLMs, using decoupled se... | 204 |
| [experience-driven-multi-agent-systems-training-fre](skills/experience-driven-multi-agent-systems-training-fre/SKILL.md) | Build self-evolving multi-agent systems that accumulate tool-level expertise through structured interaction without mode... | 168 |
| [explainable-deepfake-detection-rl](skills/explainable-deepfake-detection-rl/SKILL.md) | Build explainable deepfake detection systems using RL-enhanced Self-Blended Images and Chain-of-Thought reasoning. Use w... | 296 |
| [extracting-recurring-vulnerabilities-black-box](skills/extracting-recurring-vulnerabilities-black-box/SKILL.md) | > | 188 |
| [farm-field-aware-resolution-intelligent](skills/farm-field-aware-resolution-intelligent/SKILL.md) | Build intelligent trigger-action automation systems using FARM's two-stage architecture: contrastive retrieval + multi-a... | 187 |
| [featurebench-benchmarking-agentic-coding](skills/featurebench-benchmarking-agentic-coding/SKILL.md) | Extract feature-level coding tasks from repositories using test-driven dependency graph tracing. Use when the user says ... | 178 |
| [fimi-domain-specific-indian-finance](skills/fimi-domain-specific-indian-finance/SKILL.md) | Build domain-specialized AI agents for Indian financial systems (UPI, NPCI, RBI) using multi-stage training pipeline pat... | 243 |
| [fin-rate-real-world-financial-analytics](skills/fin-rate-real-world-financial-analytics/SKILL.md) | Analyze SEC filings and financial disclosures using the Fin-RATE three-pathway methodology: detail-oriented reasoning wi... | 182 |
| [flyaoc-evaluating-agentic-ontology](skills/flyaoc-evaluating-agentic-ontology/SKILL.md) | Build multi-agent systems for end-to-end ontology curation from scientific literature. Applies FlyAOC's agent architectu... | 184 |
| [following-dragons-code-review-guided](skills/following-dragons-code-review-guided/SKILL.md) | Extract security-relevant signals from code review comments and translate them into fuzzer-guiding annotations using the... | 158 |
| [forest-chat-adapting-vision-language-agents](skills/forest-chat-adapting-vision-language-agents/SKILL.md) | Build LLM-orchestrated agents for bi-temporal satellite image change analysis, combining vision-language models with too... | 362 |
| [found-rl-foundation-model-enhanced-reinforcement](skills/found-rl-foundation-model-enhanced-reinforcement/SKILL.md) | Architect asynchronous VLM-enhanced RL training pipelines that decouple heavy foundation model inference from simulation... | 264 |
| [fraudshield-knowledge-graph-empowered](skills/fraudshield-knowledge-graph-empowered/SKILL.md) | Detect and defend against fraudulent content in LLM inputs using knowledge-graph-augmented analysis. Builds a fraud tact... | 269 |
| [from-consistency-complementarity-aligned](skills/from-consistency-complementarity-aligned/SKILL.md) | Build multi-modal time series analysis pipelines that align numerical data with visual plots and textual captions using ... | 230 |
| [from-data-behavior-predicting](skills/from-data-behavior-predicting/SKILL.md) | Predict unintended LLM behaviors (bias, safety regressions) from training data BEFORE fine-tuning, using the MDF (Manipu... | 209 |
| [from-features-actions-explainability](skills/from-features-actions-explainability/SKILL.md) | Diagnose and explain failures in agentic AI systems using trace-based rubric evaluation, bridging static feature attribu... | 207 |
| [from-gameplay-traces-game](skills/from-gameplay-traces-game/SKILL.md) | Reverse-engineer game mechanics from gameplay traces using a two-stage causal induction pipeline: first infer a Structur... | 211 |
| [from-passive-metric-active](skills/from-passive-metric-active/SKILL.md) | Build systems that use LLM uncertainty as an active control signal -- routing computation, triggering tool calls, enabli... | 268 |
| [from-pragmas-partners-symbiotic](skills/from-pragmas-partners-symbiotic/SKILL.md) | Agentic High-Level Synthesis (HLS) optimization: autonomously analyze, insert, and tune C/C++ HLS pragmas (pipeline, unr... | 186 |
| [from-prompt-response-goal-directed-systems](skills/from-prompt-response-goal-directed-systems/SKILL.md) | Design production-grade agentic AI architectures with separated cognition/execution layers, typed tool interfaces, multi... | 177 |
| [from-sparse-decisions-dense](skills/from-sparse-decisions-dense/SKILL.md) | Build content moderation and safety classification systems using multi-attribute trajectory reasoning instead of binary ... | 261 |
| [from-task-solving-robust](skills/from-task-solving-robust/SKILL.md) | Build LLM agent workflows that stay robust under partial observability, noisy signals, shifting environments, and intern... | 199 |
| [from-utterance-vividity-training](skills/from-utterance-vividity-training/SKILL.md) | Train expressive subtitle translation LLMs using Adaptive Local Preference Optimization (ALPO) — a segment-level prefere... | 257 |
| [fullstack-agent-enhancing-agentic-fullstack](skills/fullstack-agent-enhancing-agentic-fullstack/SKILL.md) | Build production-grade full-stack web applications using a three-agent pipeline (Planning, Backend, Frontend) with devel... | 146 |
| [gender-race-bias-consumer](skills/gender-race-bias-consumer/SKILL.md) | Audit LLM-generated product recommendations for gender and race bias using marked words analysis, SVM classification, an... | 255 |
| [generalizable-interpretable-rf-fingerprinting](skills/generalizable-interpretable-rf-fingerprinting/SKILL.md) | Build RF fingerprinting systems that combine learnable 2D shapelets with pre-trained LLMs for wireless device authentica... | 168 |
| [genius-generative-fluid-intelligence](skills/genius-generative-fluid-intelligence/SKILL.md) | Evaluate and improve generative AI outputs for fluid intelligence tasks -- pattern induction from context, ad-hoc constr... | 247 |
| [graph-anchored-knowledge-indexing-retrieval-augmen](skills/graph-anchored-knowledge-indexing-retrieval-augmen/SKILL.md) | Build iterative RAG pipelines that construct evolving knowledge graphs to anchor retrieval across multiple hops. Use whe... | 221 |
| [graph-based-agent-memory-taxonomy](skills/graph-based-agent-memory-taxonomy/SKILL.md) | Design and implement graph-based memory systems for LLM agents following the extraction-storage-retrieval-evolution life... | 279 |
| [graphagents-knowledge-graph-guided-agentic](skills/graphagents-knowledge-graph-guided-agentic/SKILL.md) | Build multi-agent pipelines that use knowledge graphs to guide LLM reasoning across domains. Agents specialize in proble... | 185 |
| [graphseek-next-generation-graph-analytics](skills/graphseek-next-generation-graph-analytics/SKILL.md) | Build LLM-powered graph analytics systems using the GraphSeek two-plane architecture: a Semantic Catalog for planning ov... | 151 |
| [gutenocr-grounded-vision-language-front-end](skills/gutenocr-grounded-vision-language-front-end/SKILL.md) | Build grounded OCR pipelines using GutenOCR's prompt-based interface for reading, detection, and spatial grounding on do... | 210 |
| [harmoni-multimodal-personalization-multi-user](skills/harmoni-multimodal-personalization-multi-user/SKILL.md) | Build multi-user personalization pipelines with per-user profile tracking, multimodal perception, and LLM-driven context... | 197 |
| [harnessing-precision-querying-retrieval-augmented](skills/harnessing-precision-querying-retrieval-augmented/SKILL.md) | LLM-driven precision querying of structured tabular data via Python/Pandas code generation and retrieval-augmented extra... | 163 |
| [hidden-licensing-risks-llmware](skills/hidden-licensing-risks-llmware/SKILL.md) | Detect license incompatibilities across LLM supply chains (OSS repos, models, datasets) using the LiAgent multi-agent ex... | 182 |
| [how-decoder-only-perceive-users](skills/how-decoder-only-perceive-users/SKILL.md) | Implement Gradient-Guided Soft Masking (GGSM) attention strategies for adapting decoder-only LLMs to user representation... | 235 |
| [how-much-reasoning-retrieval-augmented](skills/how-much-reasoning-retrieval-augmented/SKILL.md) | Build contamination-aware hybrid RAG evaluation pipelines that couple knowledge graphs with text retrieval for multi-hop... | 178 |
| [hugrag-hierarchical-causal-knowledge](skills/hugrag-hierarchical-causal-knowledge/SKILL.md) | Build hierarchical causal knowledge graphs for RAG pipelines that suppress spurious correlations and enable cross-docume... | 168 |
| [hybrid-supervised-llm-pipeline-actionable-suggesti](skills/hybrid-supervised-llm-pipeline-actionable-suggesti/SKILL.md) | Build hybrid classifier-then-LLM pipelines to extract actionable suggestions from unstructured customer reviews. Use whe... | 193 |
| [hyperoffload-graph-driven-hierarchical-memory](skills/hyperoffload-graph-driven-hierarchical-memory/SKILL.md) | Design and implement compiler-driven hierarchical memory offloading for LLM inference and training on multi-tier memory ... | 226 |
| [ic-eo-interpretable-code-based-assistant](skills/ic-eo-interpretable-code-based-assistant/SKILL.md) | Build conversational Earth Observation agents that turn natural-language queries into executable, auditable Python workf... | 215 |
| [icl-evader-zero-query-black-box-evasion](skills/icl-evader-zero-query-black-box-evasion/SKILL.md) | Harden ICL classification prompts against zero-query black-box evasion attacks. Audit in-context learning pipelines for ... | 251 |
| [iesr-mcts-based-modular-reasoning](skills/iesr-mcts-based-modular-reasoning/SKILL.md) | Convert natural language questions into SQL queries using MCTS-based modular reasoning inspired by the IESR framework. D... | 242 |
| [improve-systems-user-logs](skills/improve-systems-user-logs/SKILL.md) | Implement the UNO (User log-driveN Optimization) framework to improve LLM-powered systems by distilling user interaction... | 201 |
| [improving-user-privacy-personalized](skills/improving-user-privacy-personalized/SKILL.md) | Build privacy-preserving personalized LLM systems using the P³ (Private Personalized Prediction) client-server architect... | 192 |
| [industrialized-deception-collateral-effects](skills/industrialized-deception-collateral-effects/SKILL.md) | Analyze content for AI-generated misinformation signals using the JudgeGPT/RogueGPT experimental pipeline. Evaluate text... | 216 |
| [inficoevalchain-blockchain-based-decentralized-fra](skills/inficoevalchain-blockchain-based-decentralized-fra/SKILL.md) | Design and implement decentralized LLM evaluation systems using blockchain-based consensus, multi-node validation, and s... | 202 |
| [innovator-vl-multimodal-scientific-discovery](skills/innovator-vl-multimodal-scientific-discovery/SKILL.md) | Build data-efficient multimodal scientific ML pipelines using Innovator-VL's principled training methodology. Applies tr... | 247 |
| [instructtime-time-series-classification-multimodal](skills/instructtime-time-series-classification-multimodal/SKILL.md) | Reformulate time series classification as a multimodal generative task using LLMs. Discretizes time series into tokens, ... | 165 |
| [internalizing-multi-agent-reasoning-accurate](skills/internalizing-multi-agent-reasoning-accurate/SKILL.md) | Distill multi-agent reasoning into a single efficient model for recommendation or retrieval. Use when: 'build a recommen... | 174 |
| [internalizing-reasoning-discovery-replay](skills/internalizing-reasoning-discovery-replay/SKILL.md) | Apply the STIR (Self-Distilled Tools for Internal Reasoning) pattern to build systems that discover reusable reasoning p... | 253 |
| [interpreting-agentic-systems-beyond](skills/interpreting-agentic-systems-beyond/SKILL.md) | Audit and instrument agentic AI systems for system-level interpretability and accountability. Embeds traceability, causa... | 329 |
| [interpreting-controlling-reasoning-integrated](skills/interpreting-controlling-reasoning-integrated/SKILL.md) | Interpret and control LLM reasoning behavior using Integrated Policy Gradient (IPG) attribution. Identifies which intern... | 209 |
| [isd-agent-bench-comprehensive-benchmark-evaluating](skills/isd-agent-bench-comprehensive-benchmark-evaluating/SKILL.md) | Build and evaluate LLM-based Instructional Design agents using the ADDIE framework, Context Matrix scenario generation, ... | 210 |
| [issueguard-real-time-secret-leak](skills/issueguard-real-time-secret-leak/SKILL.md) | Scan text for leaked secrets using a two-stage pipeline: regex candidate extraction followed by contextual classificatio... | 268 |
| [iterative-refinement-improves-compositional](skills/iterative-refinement-improves-compositional/SKILL.md) | Implement iterative critic-guided refinement loops for compositional image generation. Uses a VLM critic to progressivel... | 199 |
| [jade-bridging-strategic-operational-gap](skills/jade-bridging-strategic-operational-gap/SKILL.md) | Build jointly-optimized agentic RAG pipelines using the JADE pattern: a central planner co-adapted with specialized exec... | 248 |
| [jobresqa-benchmark-machine-reading](skills/jobresqa-benchmark-machine-reading/SKILL.md) | Build and evaluate multilingual machine reading comprehension systems for HR documents (resumes and job descriptions). I... | 152 |
| [joint-continual-learning-local](skills/joint-continual-learning-local/SKILL.md) | Implement DA-GRPO (Dual-Advantage Group Relative Policy Optimization) for jointly training local small language models w... | 270 |
| [just-in-time-reinforcement-learning-continual](skills/just-in-time-reinforcement-learning-continual/SKILL.md) | Implement JitRL-style continual learning for LLM agents: training-free policy optimization via experience memory, advant... | 203 |
| [kid-knowledge-injected-dual-head-learning](skills/kid-knowledge-injected-dual-head-learning/SKILL.md) | Build knowledge-grounded multimodal content classifiers using the KID dual-head architecture: entity-anchored knowledge ... | 185 |
| [knowledge-graphs-implicit-reward](skills/knowledge-graphs-implicit-reward/SKILL.md) | Build compositional reasoning systems that use knowledge graph paths as reward signals to ground LLM reasoning in verifi... | 168 |
| [koral-knowledge-graph-guided](skills/koral-knowledge-graph-guided/SKILL.md) | Build Knowledge Graph-guided LLM reasoning pipelines for operational telemetry analysis. Combines a Literature KG (extra... | 247 |
| [large-geolocation-extraction-humanitarian](skills/large-geolocation-extraction-humanitarian/SKILL.md) | Extract and geocode location mentions from humanitarian and crisis texts using a two-step LLM pipeline: few-shot NER for... | 213 |
| [large-scale-multidimensional-knowledge-profiling](skills/large-scale-multidimensional-knowledge-profiling/SKILL.md) | Build multidimensional profiling pipelines for large scientific paper corpora. Combines BERTopic clustering, LLM-structu... | 246 |
| [latentchem-textual-cot-latent](skills/latentchem-textual-cot-latent/SKILL.md) | Apply LatentChem's latent-space reasoning paradigm to chemical computation tasks -- replacing verbose textual Chain-of-T... | 189 |
| [learning-compose-cross-domain-agentic](skills/learning-compose-cross-domain-agentic/SKILL.md) | Generate cross-domain agentic workflows using decompose-recompose-decide composition over reusable capability bases. Use... | 159 |
| [learning-decentralized-collaboration-multi-agent](skills/learning-decentralized-collaboration-multi-agent/SKILL.md) | Design and orchestrate decentralized multi-LLM collaboration systems using Multi-Agent Actor-Critic (MAAC) patterns from... | 218 |
| [learning-decode-against-compositional](skills/learning-decode-against-compositional/SKILL.md) | Detect and mitigate compositional hallucinations in video multimodal LLM outputs using triple-pathway contrastive decodi... | 284 |
| [learning-irrecoverable-error-localized-policy](skills/learning-irrecoverable-error-localized-policy/SKILL.md) | Debug multi-step tool-using agent pipelines by localizing the first irrecoverable error via binary-search rollback, then... | 175 |
| [lec-kg-llm-embedding-collaborative-framework](skills/lec-kg-llm-embedding-collaborative-framework/SKILL.md) | Build domain-specific knowledge graphs from unstructured text using an iterative LLM + embedding validation loop. Combin... | 186 |
| [legalmalr-multi-agent-query-understanding](skills/legalmalr-multi-agent-query-understanding/SKILL.md) | Multi-agent query reformulation and LLM reranking for retrieval over legal, regulatory, or domain-specific corpora. Use ... | 168 |
| [legalone-family-foundation-reliable](skills/legalone-family-foundation-reliable/SKILL.md) | Build domain-specialized LLM training pipelines using the LegalOne three-phase methodology: Plasticity-Adjusted Sampling... | 259 |
| [lemon-agent-technical-report](skills/lemon-agent-technical-report/SKILL.md) | Orchestrate multi-agent workflows using the Lemon Agent orchestrator-worker pattern with hierarchical scheduling, progre... | 186 |
| [lemur-corpus-robust-fine-tuning](skills/lemur-corpus-robust-fine-tuning/SKILL.md) | Build multilingual legal document retrieval systems by fine-tuning embedding models on domain-specific corpora with cont... | 234 |
| [less-enough-synthesizing-diverse](skills/less-enough-synthesizing-diverse/SKILL.md) | Synthesize maximally diverse training data for LLM post-training using Feature Activation Coverage (FAC). Identifies mis... | 181 |
| [leveraging-data-say-no](skills/leveraging-data-say-no/SKILL.md) | Implement memory-augmented selective prediction for vision-language models using retrieval-based confidence scoring and ... | 194 |
| [leveraging-turkish-skill-extraction](skills/leveraging-turkish-skill-extraction/SKILL.md) | Extract and normalize skills from job postings using a two-stage LLM pipeline: dynamic few-shot skill identification fol... | 198 |
| [linglanmidian-systematic-evaluation-tcm](skills/linglanmidian-systematic-evaluation-tcm/SKILL.md) | Build rigorous, multi-task evaluation benchmarks for domain-specific LLMs using the LingLanMiDian methodology: synonym-t... | 245 |
| [live-evo-online-evolution-agentic](skills/live-evo-online-evolution-agentic/SKILL.md) | Implement online self-evolving memory for LLM agents using dual-bank architecture (Experience Bank + Meta-Guideline Bank... | 204 |
| [livemedbench-contamination-free-medical-benchmark](skills/livemedbench-contamination-free-medical-benchmark/SKILL.md) | Build contamination-free LLM evaluation pipelines with multi-agent data curation and automated rubric-based scoring. Use... | 296 |
| [livibench-omnimodal-benchmark-interactive](skills/livibench-omnimodal-benchmark-interactive/SKILL.md) | Build omnimodal benchmarks and evaluation pipelines for interactive video understanding (livestreams, real-time comments... | 238 |
| [llamea-sage-guiding-automated-algorithm](skills/llamea-sage-guiding-automated-algorithm/SKILL.md) | Guide LLM-driven algorithm generation using AST structural feedback and explainable AI. Extracts graph-theoretic and com... | 226 |
| [llm-autodp-automatic-data-processing](skills/llm-autodp-automatic-data-processing/SKILL.md) | Automatically generate and optimize data processing pipelines for LLM fine-tuning datasets using an agent-driven iterati... | 168 |
| [llm-based-sql-generation-prompting](skills/llm-based-sql-generation-prompting/SKILL.md) | Generate accurate SQL from natural language using the SSEV pipeline: schema-linked prompting, execution-guided self-refi... | 174 |
| [llm-not-all-you](skills/llm-not-all-you/SKILL.md) | Systematic model selection advisor for classification tasks — chooses between classical ML, zero-shot LLMs/VLMs, and fin... | 187 |
| [lmmrec-llm-driven-motivation-aware-multimodal](skills/lmmrec-llm-driven-motivation-aware-multimodal/SKILL.md) | Build motivation-aware recommendation systems that use LLM chain-of-thought prompting to extract user and item motivatio... | 217 |
| [logicscore-fine-grained-logic-evaluation](skills/logicscore-fine-grained-logic-evaluation/SKILL.md) | Evaluate the logical integrity of LLM-generated multi-hop answers using Horn Rule backward chaining. Scores Completeness... | 185 |
| [longcat-flash-thinking-2601-technical-report](skills/longcat-flash-thinking-2601-technical-report/SKILL.md) | Build robust multi-tool agentic pipelines with noise-aware execution, parallel reasoning, and environment scaling patter... | 311 |
| [mad-modality-adaptive-decoding-mitigating](skills/mad-modality-adaptive-decoding-mitigating/SKILL.md) | Implement Modality-Adaptive Decoding (MAD) to suppress cross-modal hallucinations in multimodal LLMs. Uses self-assessme... | 227 |
| [made-benchmark-environments-closed-loop](skills/made-benchmark-environments-closed-loop/SKILL.md) | Build closed-loop discovery benchmarks where an agent iteratively proposes, evaluates, and refines candidates under a fi... | 144 |
| [marble-multi-agent-reasoning-bioinformatics](skills/marble-multi-agent-reasoning-bioinformatics/SKILL.md) | Iteratively refine bioinformatics and ML models using MARBLE's multi-agent debate framework with role-specialized agents... | 206 |
| [markovscale-optimal-sequential-scaling](skills/markovscale-optimal-sequential-scaling/SKILL.md) | Implement MarkovScale's principled sequential scaling for LLM inference pipelines. Models retry/refinement loops as a tw... | 193 |
| [mas-prove-understanding-process-verification](skills/mas-prove-understanding-process-verification/SKILL.md) | Design and implement process verification for multi-agent LLM systems. Add intermediate-step evaluation to multi-agent w... | 237 |
| [masalbench-benchmark-contextual-cross-cultural](skills/masalbench-benchmark-contextual-cross-cultural/SKILL.md) | Build cross-cultural figurative language benchmarks and evaluation pipelines for LLMs. Applies the MasalBench methodolog... | 186 |
| [mata-multiagent-framework-for](skills/mata-multiagent-framework-for/SKILL.md) | Multi-agent table question answering using MATA's three-path reasoning strategy (Chain-of-Thought, Program-of-Thought, T... | 167 |
| [mata-trainable-hierarchical-automaton](skills/mata-trainable-hierarchical-automaton/SKILL.md) | Build multi-agent visual reasoning systems using hierarchical finite-state automata with a trainable hyper agent that or... | 303 |
| [medbeads-agent-native-immutable-data](skills/medbeads-agent-native-immutable-data/SKILL.md) | Build immutable, agent-native medical data pipelines using Merkle DAG structures (MedBeads pattern). Converts mutable EM... | 182 |
| [medmo-grounding-understanding-multimodal](skills/medmo-grounding-understanding-multimodal/SKILL.md) | Build medical image analysis pipelines with multi-stage grounded reasoning: cross-modal alignment, instruction-tuned VQA... | 313 |
| [medspeak-knowledge-graph-aided-asr](skills/medspeak-knowledge-graph-aided-asr/SKILL.md) | Build knowledge-graph-aided ASR error correction pipelines for medical speech, using phonetic similarity + semantic retr... | 262 |
| [memcast-memory-driven-time-series](skills/memcast-memory-driven-time-series/SKILL.md) | Build memory-augmented time series forecasting systems using hierarchical experience storage (historical patterns, reaso... | 196 |
| [mermaid-memory-enhanced-retrieval-reasoning](skills/mermaid-memory-enhanced-retrieval-reasoning/SKILL.md) | Memory-enhanced multi-agent retrieval and reasoning for veracity assessment and fact-checking. Use when: 'verify this cl... | 189 |
| [mhdash-online-platform-benchmarking](skills/mhdash-online-platform-benchmarking/SKILL.md) | Build risk-aware evaluation pipelines for mental health AI assistants using the MHDash framework. Implements multi-dimen... | 285 |
| [mind-ambiguity-aleatoric-uncertainty](skills/mind-ambiguity-aleatoric-uncertainty/SKILL.md) | Detect ambiguous user queries in safety-critical QA systems using aleatoric uncertainty probes on LLM hidden states, the... | 225 |
| [mirror-multi-agent-framework-iterative](skills/mirror-multi-agent-framework-iterative/SKILL.md) | Translate natural language optimization problems into mathematical models and solver code using MIRROR's multi-agent pip... | 166 |
| [mixing-expert-knowledge-bring](skills/mixing-expert-knowledge-bring/SKILL.md) | Integrate domain expert knowledge into LLM fine-tuning pipelines using mixed cold-start SFT and reinforcement learning. ... | 199 |
| [mmts-bench-comprehensive-benchmark-time](skills/mmts-bench-comprehensive-benchmark-time/SKILL.md) | Evaluate and improve LLM performance on time series question-answering using the MMTS-BENCH hierarchical taxonomy. Cover... | 193 |
| [moco-one-stop-shop-collaboration](skills/moco-one-stop-shop-collaboration/SKILL.md) | Design and implement multi-LM collaboration pipelines using the MoCo framework's 26 methods across four collaboration le... | 225 |
| [more-than-quick-glance](skills/more-than-quick-glance/SKILL.md) | Implement LASER-KV-style KV-cache compression for LLM inference pipelines using block-wise accumulative budgeting and hy... | 256 |
| [mpib-benchmark-medical-prompt](skills/mpib-benchmark-medical-prompt/SKILL.md) | Evaluate and defend clinical LLM systems against prompt injection attacks using the MPIB benchmark methodology. Implemen... | 177 |
| [mrag-benchmarking-retrieval-augmented-generation](skills/mrag-benchmarking-retrieval-augmented-generation/SKILL.md) | Build and evaluate biomedical RAG pipelines using the MRAG benchmark methodology. Configures retrieval, prompting, and g... | 183 |
| [multi-agent-causal-reasoning-system](skills/multi-agent-causal-reasoning-system/SKILL.md) | Build multi-agent systems that discover causal rules from event sequences using specialized agents (causal discovery, co... | 225 |
| [multi-agent-collaborative-intrusion-detection](skills/multi-agent-collaborative-intrusion-detection/SKILL.md) | Build multi-agent intrusion detection systems using LLM-enhanced collaborative agents for network traffic classification... | 309 |
| [multi-agent-constraint-factorization-reveals](skills/multi-agent-constraint-factorization-reveals/SKILL.md) | Orchestrate multi-agent LLM pipelines using constraint factorization -- decomposing complex requirements into separate c... | 157 |
| [multi-agent-end-to-end-vulnerability-management](skills/multi-agent-end-to-end-vulnerability-management/SKILL.md) | Detect, confirm, repair, and validate recurring software vulnerabilities using a multi-agent pipeline modeled on MAVM. B... | 196 |
| [multi-targeted-graph-backdoor-attack](skills/multi-targeted-graph-backdoor-attack/SKILL.md) | Implement and analyze multi-targeted backdoor attacks on Graph Neural Networks (GNNs) using subgraph injection. Use when... | 193 |
| [multimodal-fine-tuning-synthetic-captions](skills/multimodal-fine-tuning-synthetic-captions/SKILL.md) | Generate synthetic image captions with MLLMs and fine-tune CLIP models using multimodal objectives with supervised contr... | 201 |
| [multimodal-multi-agent-ransomware-analysis](skills/multimodal-multi-agent-ransomware-analysis/SKILL.md) | Build multimodal multi-agent pipelines for ransomware classification using specialized per-modality agents, autoencoder ... | 261 |
| [multivis-agent-multi-agent-framework-logic](skills/multivis-agent-multi-agent-framework-logic/SKILL.md) | Build reliable multi-agent data visualization pipelines with logic rule constraints. Use when: 'generate a chart from th... | 193 |
| [next-gen-captchas-leveraging-cognitive](skills/next-gen-captchas-leveraging-cognitive/SKILL.md) | Design and implement AI-resistant CAPTCHA systems that exploit the cognitive gap between humans and GUI agents. Covers p... | 253 |
| [note2chat-improving-multi-turn-clinical](skills/note2chat-improving-multi-turn-clinical/SKILL.md) | Build structured multi-turn clinical history-taking agents and diagnostic chatbots using the Note2Chat framework: conver... | 175 |
| [now-you-hear-me](skills/now-you-hear-me/SKILL.md) | Audit and defend large audio-language models (LALMs) against narrative-style audio jailbreaks. Based on the 'Now You Hea... | 311 |
| [omni-rrm-advancing-omni-reward](skills/omni-rrm-advancing-omni-reward/SKILL.md) | Build rubric-grounded reward models and preference evaluation pipelines for multimodal AI outputs. Use when asked to 'ev... | 180 |
| [omni-safety-under-cross-modality-conflict](skills/omni-safety-under-cross-modality-conflict/SKILL.md) | Audit and harden omni-modal LLM safety against cross-modality attacks using refusal-vector analysis and OmniSteer alignm... | 217 |
| [omnirag-agent-agentic-omnimodal-reasoning](skills/omnirag-agent-agentic-omnimodal-reasoning/SKILL.md) | Build agentic multimodal RAG pipelines that answer questions over long audio-video content under resource constraints. U... | 278 |
| [omnireview-large-scale-benchmark-llm-enhanced](skills/omnireview-large-scale-benchmark-llm-enhanced/SKILL.md) | Build reviewer/expert recommendation systems using LLM-generated semantic profiles and Multi-gate Mixture-of-Experts (MM... | 204 |
| [on-use-generate-dataset](skills/on-use-generate-dataset/SKILL.md) | Generate diverse, validated datasets of neural network implementations using LLM-driven combinatorial design. Use when: ... | 195 |
| [on-use-support-conduction](skills/on-use-support-conduction/SKILL.md) | LLM-assisted systematic literature review and mapping study pipeline. Automates screening, data extraction, and classifi... | 167 |
| [one-size-many-fits](skills/one-size-many-fits/SKILL.md) | Build group-aware advertising image generation systems that align diverse user-segment click preferences instead of opti... | 236 |
| [ontology-to-tools-compilation-executable-semantic-](skills/ontology-to-tools-compilation-executable-semantic/SKILL.md) | Compile domain ontologies (OWL/RDFS/JSON-LD schemas) into executable tool interfaces with embedded semantic constraints,... | 192 |
| [openguandan-large-scale-imperfect-information](skills/openguandan-large-scale-imperfect-information/SKILL.md) | Build AI agents for the OpenGuanDan imperfect-information card game benchmark. Covers WebSocket client implementation, g... | 366 |
| [opportunities-aiml-rubin-lsst](skills/opportunities-aiml-rubin-lsst/SKILL.md) | Build trustworthy ML pipelines for large-scale scientific data analysis with calibrated uncertainties, simulation-based ... | 249 |
| [optimizing-small-sample-experience-learning-llm-ba](skills/optimizing-small-sample-experience-learning-llm-ba/SKILL.md) | Implement the ExperienceWeaver hierarchical experience-learning framework to improve text quality from small feedback se... | 195 |
| [orthogonal-hierarchical-decomposition-structure-aw](skills/orthogonal-hierarchical-decomposition-structure-aw/SKILL.md) | Decompose complex tables with multi-level headers, merged cells, and irregular layouts into orthogonal column/row trees ... | 231 |
| [paperbanana-automating-academic-illustration](skills/paperbanana-automating-academic-illustration/SKILL.md) | Generate publication-ready academic illustrations using a multi-agent pipeline inspired by PaperBanana. Orchestrates ret... | 197 |
| [papersearchqa-learning-search-reason](skills/papersearchqa-learning-search-reason/SKILL.md) | Build iterative search-and-reason agents for scientific literature QA. Uses the PaperSearchQA pattern: interleaved think... | 233 |
| [parse-open-domain-reasoning-question](skills/parse-open-domain-reasoning-question/SKILL.md) | Build and evaluate reasoning-focused QA systems for low-resource languages using the PARSE methodology: structured promp... | 220 |
| [pathreasoner-r1-instilling-structured-reasoning](skills/pathreasoner-r1-instilling-structured-reasoning/SKILL.md) | Build knowledge-graph-guided structured reasoning pipelines for vision-language models in computational pathology. Imple... | 296 |
| [pcbschemagen-constraint-guided-schematic-design](skills/pcbschemagen-constraint-guided-schematic-design/SKILL.md) | Generate PCB schematics from natural language using constraint-guided LLM code generation with knowledge-graph verificat... | 209 |
| [pcl-reasoner-v15-advancing-math-reasoning-offline](skills/pcl-reasoner-v15-advancing-math-reasoning-offline/SKILL.md) | Implement offline reinforcement learning pipelines for LLM reasoning tasks — decoupling data collection from training fo... | 255 |
| [pearl-plan-exploration-adaptive](skills/pearl-plan-exploration-adaptive/SKILL.md) | Apply PEARL's two-phase tool orchestration: offline tool exploration to learn valid usage patterns and failure modes, th... | 172 |
| [peerrank-autonomous-evaluation-web-grounded](skills/peerrank-autonomous-evaluation-web-grounded/SKILL.md) | Implement PeerRank-style autonomous multi-model evaluation pipelines where LLMs symmetrically generate tasks, answer wit... | 215 |
| [persodpo-scalable-preference-optimization](skills/persodpo-scalable-preference-optimization/SKILL.md) | Build scalable preference optimization pipelines for persona-grounded dialogue systems using multi-LLM evaluation. Use w... | 173 |
| [phostream-benchmarking-real-world-streaming](skills/phostream-benchmarking-real-world-streaming/SKILL.md) | Build streaming multimodal benchmarks and evaluate omnimodal assistants on continuous audio-visual input with temporal r... | 238 |
| [physprover-advancing-automatic-theorem](skills/physprover-advancing-automatic-theorem/SKILL.md) | Build formal theorem proving pipelines for physics and scientific domains using conjecture-based data generation, Lean 4... | 218 |
| [planner-auditor-twin-agentic-discharge](skills/planner-auditor-twin-agentic-discharge/SKILL.md) | Implement a Planner-Auditor twin architecture that decouples LLM generation from deterministic validation with self-impr... | 189 |
| [polarmem-training-free-polarized-latent](skills/polarmem-training-free-polarized-latent/SKILL.md) | Build polarized memory systems for multimodal agents that encode both positive and negative evidence as graph constraint... | 183 |
| [precise-reducing-bias-evaluations](skills/precise-reducing-bias-evaluations/SKILL.md) | Implement the PRECISE framework to debias LLM-as-judge evaluations of search, ranking, and RAG systems by combining a sm... | 212 |
| [predicting-intermittent-job-failure](skills/predicting-intermittent-job-failure/SKILL.md) | Classify and diagnose intermittent CI/CD job failures from execution logs using the FlaXifyer few-shot approach and LogS... | 280 |
| [prism-xr-empowering-privacy-aware-xr](skills/prism-xr-empowering-privacy-aware-xr/SKILL.md) | Build privacy-aware pipelines that filter sensitive content from visual frames before sending to cloud AI models, using ... | 301 |
| [privacy-collapse-benign-fine-tuning](skills/privacy-collapse-benign-fine-tuning/SKILL.md) | Audit fine-tuning datasets and pipelines for privacy collapse — the silent failure where benign training data degrades a... | 195 |
| [prograph-r1-progress-aware-reinforcement-learning](skills/prograph-r1-progress-aware-reinforcement-learning/SKILL.md) | Build progress-aware GraphRAG agents that traverse knowledge graphs with structure-aware hypergraph retrieval and dense ... | 167 |
| [protean-compiler-agile-framework](skills/protean-compiler-agile-framework/SKILL.md) | Guide fine-grained LLVM compiler phase ordering using the Protean framework's agile optimization approach — clustering p... | 149 |
| [protoken-token-level-attribution-federated](skills/protoken-token-level-attribution-federated/SKILL.md) | Implement ProToken-style token-level attribution to trace which federated learning client(s) contributed to each generat... | 164 |
| [proxywar-dynamic-assessment-of](skills/proxywar-dynamic-assessment-of/SKILL.md) | Build competitive game-arena evaluation frameworks for LLM-generated code using ProxyWar's multi-layer pipeline: agent g... | 197 |
| [puda-private-user-dataset](skills/puda-private-user-dataset/SKILL.md) | Build privacy-preserving personalized AI systems using Puda's multi-granularity user data architecture. Implements clien... | 213 |
| [quasar-universal-autonomous-system](skills/quasar-universal-autonomous-system/SKILL.md) | Build autonomous multi-scale scientific simulation pipelines using the QUASAR architecture: a Strategist-Operator-Evalua... | 165 |
| [query-efficient-agentic-graph-extraction](skills/query-efficient-agentic-graph-extraction/SKILL.md) | > | 239 |
| [r1-syntheticvl-synthetic-data-generative](skills/r1-syntheticvl-synthetic-data-generative/SKILL.md) | Synthesize high-quality multimodal training data using Collective Adversarial Data Synthesis (CADS). Implements a cyclic... | 226 |
| [ragturk-best-practices-retrieval](skills/ragturk-best-practices-retrieval/SKILL.md) | Design and optimize RAG pipelines for Turkish and other morphologically rich languages (Turkish, Finnish, Hungarian, Kor... | 208 |
| [raicl-retrieval-augmented-in-context-learning](skills/raicl-retrieval-augmented-in-context-learning/SKILL.md) | Build retrieval-augmented in-context learning (RAICL) pipelines that convert time-series or signal data into images and ... | 228 |
| [rapid-real-time-deterministic-trajectory](skills/rapid-real-time-deterministic-trajectory/SKILL.md) | Distill diffusion-based trajectory planners into fast deterministic policies using score-regularized optimization and sa... | 185 |
| [rapo-risk-aware-preference-optimization](skills/rapo-risk-aware-preference-optimization/SKILL.md) | Apply risk-aware preference optimization to make LLM reasoning chains safer against jailbreak attacks. Implements adapti... | 203 |
| [rate-reviewer-profiling-annotation-free](skills/rate-reviewer-profiling-annotation-free/SKILL.md) | | | 224 |
| [rc-grpo-reward-conditioned-group-relative](skills/rc-grpo-reward-conditioned-group-relative/SKILL.md) | Implement reward-conditioned training pipelines for multi-turn tool-calling agents using RC-GRPO. Injects discrete rewar... | 228 |
| [realhd-high-quality-dataset-robust](skills/realhd-high-quality-dataset-robust/SKILL.md) | Detect AI-generated images using NLM noise entropy analysis and build robust forensic detection pipelines. Use when: 'de... | 230 |
| [realistic-synthetic-household-data](skills/realistic-synthetic-household-data/SKILL.md) | Generate realistic synthetic household datasets with bidirectional persona-environment coupling for embodied AI training... | 179 |
| [reasoning-augmented-representations-multimodal-ret](skills/reasoning-augmented-representations-multimodal-ret/SKILL.md) | Decouple reasoning from embedding compression in multimodal retrieval pipelines by enriching queries and corpus entries ... | 223 |
| [reasoning-tool-use-compete-agentic](skills/reasoning-tool-use-compete-agentic/SKILL.md) | Diagnose and fix interference between reasoning and tool-use in agentic AI systems using LEAS attribution and DART-style... | 204 |
| [redvisor-reasoning-aware-prompt-injection](skills/redvisor-reasoning-aware-prompt-injection/SKILL.md) | Defend LLM applications against prompt injection using RedVisor's two-phase reasoning-then-responding architecture. Impl... | 223 |
| [refer-agent-collaborative-multi-agent-system](skills/refer-agent-collaborative-multi-agent-system/SKILL.md) | Build collaborative multi-agent systems that use alternating reasoning-reflection cycles with specialized agent roles, c... | 179 |
| [refuge-feature-generation-prediction](skills/refuge-feature-generation-prediction/SKILL.md) | Automated feature engineering for prediction tasks on relational databases using a multi-agent LLM pipeline. Generates, ... | 168 |
| [reinforced-attention-learning](skills/reinforced-attention-learning/SKILL.md) | Implement Reinforced Attention Learning (RAL) for multimodal LLMs — optimize attention distributions via policy gradient... | 216 |
| [remedit-diffusion-editing-riemannian](skills/remedit-diffusion-editing-riemannian/SKILL.md) | Implement Riemannian-geometry-based diffusion image editing pipelines using geodesic latent navigation, dual-SLERP blend... | 244 |
| [resagent-entropy-based-prior-point](skills/resagent-entropy-based-prior-point/SKILL.md) | Implement entropy-guided coarse-to-fine visual grounding pipelines for referring expression segmentation and point-promp... | 266 |
| [research-multi-stage-machine-learning](skills/research-multi-stage-machine-learning/SKILL.md) | Build multi-stage search pipelines that separate recall from precision for discovering datasets, documents, or resources... | 279 |
| [rethinking-genomic-modeling-optical](skills/rethinking-genomic-modeling-optical/SKILL.md) | Implement OpticalDNA-style pipelines that render DNA sequences as 2D visual layouts and process them with OCR-capable vi... | 248 |
| [rethinking-irregular-time-series](skills/rethinking-irregular-time-series/SKILL.md) | Design and implement irregular time series classification pipelines for clinical/ICU data with high missing-value rates.... | 186 |
| [rethinking-llm-as-a-judge-representation-as-a-judg](skills/rethinking-llm-as-a-judge-representation-as-a-judg/SKILL.md) | Build probing-based evaluation pipelines that judge LLM output quality using hidden-state representations from small lan... | 160 |
| [revisiting-adaptive-rounding-vectorized](skills/revisiting-adaptive-rounding-vectorized/SKILL.md) | Implement VQRound -- a parameter-efficient adaptive rounding framework for LLM post-training quantization that reparamet... | 175 |
| [revisiting-salient-object-detection](skills/revisiting-salient-object-detection/SKILL.md) | Build observer-centric salient object detection systems using the Perceive-Reflect-Adjust agentic loop. Combines a Visio... | 257 |
| [robustexplain-evaluating-robustness-llm-based](skills/robustexplain-evaluating-robustness-llm-based/SKILL.md) | Evaluate robustness of LLM-generated recommendation explanations under realistic user behavior noise. Use when: 'test ex... | 211 |
| [rpo-rag-aligning-small-relation-aware](skills/rpo-rag-aligning-small-relation-aware/SKILL.md) | Build knowledge-graph-grounded RAG pipelines that align small LLMs (under 8B params) with relation-aware preference opti... | 259 |
| [ruleflow-generating-reusable-program](skills/ruleflow-generating-reusable-program/SKILL.md) | Optimize Pandas code by discovering per-program improvements, generalizing them into reusable rewrite rules, and applyin... | 160 |
| [rulesmith-multi-agent-automated-game](skills/rulesmith-multi-agent-automated-game/SKILL.md) | Automated game balancing using multi-agent LLM self-play coupled with Bayesian optimization. Use when the user asks to '... | 193 |
| [rvcbench-benchmarking-robustness-voice](skills/rvcbench-benchmarking-robustness-voice/SKILL.md) | Benchmark and harden voice cloning systems against real-world degradation using the RVCBench framework. Evaluates VC mod... | 164 |
| [safepred-predictive-guardrail-computer-using](skills/safepred-predictive-guardrail-computer-using/SKILL.md) | Implement predictive safety guardrails for computer-using agents and automated pipelines using world-model-based risk pr... | 179 |
| [scalable-generative-game-engine](skills/scalable-generative-game-engine/SKILL.md) | Design and deploy real-time generative game engines that break the Memory Wall via hardware-algorithm co-design. Covers ... | 162 |
| [scaled-surrogate-gradient-codec-aware-learning](skills/scaled-surrogate-gradient-codec-aware-learning/SKILL.md) | Build end-to-end video processing pipelines that train learned downsamplers/upsamplers through real non-differentiable c... | 215 |
| [scidatacopilot-agentic-data-preparation](skills/scidatacopilot-agentic-data-preparation/SKILL.md) | Build agentic pipelines that ingest heterogeneous raw scientific data, parse research intent, and produce analysis-ready... | 240 |
| [se-bench-benchmarking-self-evolution-knowledge](skills/se-bench-benchmarking-self-evolution-knowledge/SKILL.md) | Design knowledge-internalization benchmarks and closed-book training pipelines for LLM self-evolution. Use when: 'build ... | 163 |
| [selective-steering-norm-preserving-control](skills/selective-steering-norm-preserving-control/SKILL.md) | Implement norm-preserving activation steering for LLMs using discriminative layer selection and Givens rotation. Use whe... | 174 |
| [self-evolving-recommendation-system-end-to-end](skills/self-evolving-recommendation-system-end-to-end/SKILL.md) | Build autonomous ML optimization pipelines that use LLM agents to generate, evaluate, and deploy model improvements in a... | 168 |
| [self-hinting-enhance-reinforcement-learning](skills/self-hinting-enhance-reinforcement-learning/SKILL.md) | Apply the SAGE self-hinting technique to improve LLM problem-solving by generating graduated hints that boost solution d... | 174 |
| [self-improving-pretraining-post-trained-pretrain](skills/self-improving-pretraining-post-trained-pretrain/SKILL.md) | Build data curation pipelines that use a strong post-trained model to score, filter, and rewrite pretraining corpora for... | 256 |
| [semanticalli-caching-reasoning-not](skills/semanticalli-caching-reasoning-not/SKILL.md) | Implement pipeline-aware intermediate representation (IR) caching for agentic systems. Instead of caching final LLM resp... | 202 |
| [seta-statistical-fault-attribution](skills/seta-statistical-fault-attribution/SKILL.md) | Diagnose and attribute faults in compound AI systems (multi-model pipelines) using SETA's modular robustness testing fra... | 247 |
| [skillrl-evolving-agents-recursive](skills/skillrl-evolving-agents-recursive/SKILL.md) | Build self-improving agent systems that distill raw execution traces into a hierarchical skill library (SkillBank) and r... | 184 |
| [small-beautiful-practical-log](skills/small-beautiful-practical-log/SKILL.md) | Build efficient log parsing systems that extract structured templates from raw log messages using a dual-cache architect... | 191 |
| [smartoracle-agentic-approach](skills/smartoracle-agentic-approach/SKILL.md) | Agentic differential oracle for triaging cross-implementation discrepancies. Decomposes bug triage into specialized sub-... | 177 |
| [snapmla-efficient-longcontext-mla](skills/snapmla-efficient-longcontext-mla/SKILL.md) | While FP8 attention has shown substantial promise in innovations like FlashAttention-3, its integration into the decodin... | 88 |
| [snapmla-long-context-mla-decoding](skills/snapmla-long-context-mla-decoding/SKILL.md) | Deploy and optimize FP8-quantized Multi-head Latent Attention (MLA) decoding for long-context LLM inference on Hopper GP... | 186 |
| [socratic-geo-synthetic-data-generation](skills/socratic-geo-synthetic-data-generation/SKILL.md) | Generate high-quality synthetic training data through multi-agent feedback loops where a Teacher agent creates parameter... | 226 |
| [sparc-rag-adaptive-sequential-parallel-scaling](skills/sparc-rag-adaptive-sequential-parallel-scaling/SKILL.md) | Implement multi-agent RAG systems with coordinated sequential-parallel scaling and shared context management for complex... | 248 |
| [sparseeval-evaluation-sparse-optimization](skills/sparseeval-evaluation-sparse-optimization/SKILL.md) | Efficiently evaluate LLMs on benchmarks by selecting a small subset of anchor items via sparse optimization, reproducing... | 221 |
| [spava-accelerating-long-video-understanding](skills/spava-accelerating-long-video-understanding/SKILL.md) | Implement Spava-style sequence-parallel approximate attention for accelerating long-video inference across multiple GPUs... | 200 |
| [spd-faith-bench-diagnosing-improving](skills/spd-faith-bench-diagnosing-improving/SKILL.md) | Diagnose and improve faithfulness of chain-of-thought reasoning in multimodal LLM pipelines using the SPD-Faith Bench me... | 237 |
| [spider-sense-intrinsic-risk-sensing](skills/spider-sense-intrinsic-risk-sensing/SKILL.md) | Implement event-driven, hierarchical security screening for LLM agent systems using Intrinsic Risk Sensing. Adds latent ... | 212 |
| [spotagent-grounding-visual-geo-localization](skills/spotagent-grounding-visual-geo-localization/SKILL.md) | Build agentic geo-localization systems that combine vision-language model reasoning with tool-assisted verification usin... | 248 |
| [st-raptor-agentic-system-semi-structured](skills/st-raptor-agentic-system-semi-structured/SKILL.md) | Agentic system for answering questions about semi-structured tables using tree-based structural modeling and multi-step ... | 222 |
| [standardizing-longitudinal-radiology-report](skills/standardizing-longitudinal-radiology-report/SKILL.md) | Build LLM-based pipelines that automatically detect and classify longitudinal (temporal) changes in radiology reports. U... | 230 |
| [state-art-llm-enabled-interaction](skills/state-art-llm-enabled-interaction/SKILL.md) | Build LLM-powered natural language interfaces for data visualization — NL2VIS pipelines, conversational chart analytics,... | 258 |
| [stateless-yet-not-forgetful](skills/stateless-yet-not-forgetful/SKILL.md) | Detect, audit, and defend against implicit memory channels in LLM-powered systems where models encode hidden state in ou... | 245 |
| [status-hierarchies](skills/status-hierarchies/SKILL.md) | Detect and mitigate status hierarchy bias in multi-agent LLM systems. Applies expectation states theory to audit deferen... | 229 |
| [steereval-framework-evaluating-steerability](skills/steereval-framework-evaluating-steerability/SKILL.md) | Evaluate and improve the steerability of natural-language-profile-based recommender systems using the SteerEval framewor... | 191 |
| [step-35-flash-open](skills/step-35-flash-open/SKILL.md) | Build efficient agentic AI systems using sparse MoE routing, hybrid sliding-window/full attention, multi-token predictio... | 226 |
| [steuerllm-local-specialized-german](skills/steuerllm-local-specialized-german/SKILL.md) | Build domain-specialized LLM pipelines for formal-rule domains (tax law, legal, regulatory) using retrieval-augmented sy... | 203 |
| [syncabel-synthetic-contextualized-augmentation](skills/syncabel-synthetic-contextualized-augmentation/SKILL.md) | Generate synthetic training data for biomedical entity linking using LLM-based contextualized augmentation. Use when: 'g... | 193 |
| [synthagent-multi-agent-framework-realistic](skills/synthagent-multi-agent-framework-realistic/SKILL.md) | Build multi-agent pipelines that generate realistic synthetic patient profiles by integrating epidemiological data, medi... | 298 |
| [system-name-address-parsing](skills/system-name-address-parsing/SKILL.md) | Parse unstructured person names and addresses into a structured 17-field schema using prompt-driven extraction with laye... | 207 |
| [t-llm-teaching-forecast-time](skills/t-llm-teaching-forecast-time/SKILL.md) | Implement temporal distillation pipelines that teach LLMs to forecast time series by training a lightweight trend+freque... | 155 |
| [tamperbench-systematically-stress-testing-safety](skills/tamperbench-systematically-stress-testing-safety/SKILL.md) | Set up and run TamperBench pipelines to systematically stress-test LLM safety under fine-tuning and tampering attacks. U... | 250 |
| [task-oriented-robot-human-handovers-legged](skills/task-oriented-robot-human-handovers-legged/SKILL.md) | Implement task-oriented robot-to-human object handover systems using LLM-driven affordance reasoning and exemplar-based ... | 261 |
| [teaching-evaluating-reason-about](skills/teaching-evaluating-reason-about/SKILL.md) | Apply knowledge-augmented reasoning distillation for polymer design tasks. Builds structured Chain-of-Thought pipelines ... | 202 |
| [temp-r1-unified-autonomous-agent](skills/temp-r1-unified-autonomous-agent/SKILL.md) | Build autonomous agents that answer complex temporal questions over knowledge graphs or time-stamped datasets using stru... | 200 |
| [text-summarization-global-structure](skills/text-summarization-global-structure/SKILL.md) | Summarize long documents while preserving global semantic structure and logical coherence using topology-guided pruning ... | 165 |
| [the-clef-2026-finmmeval-lab](skills/the-clef-2026-finmmeval-lab/SKILL.md) | Build multilingual, multimodal financial AI evaluation pipelines using the FinMMEval framework. Covers financial exam QA... | 246 |
| [the-effectiveness-style-vectors](skills/the-effectiveness-style-vectors/SKILL.md) | Implement activation steering with style vectors to control LLM emotional tone at inference time. Compute contrastive ac... | 276 |
| [the-landscape-prompt-injection](skills/the-landscape-prompt-injection/SKILL.md) | Harden LLM agent systems against prompt injection using layered text/model/execution defenses and the AgentPI evaluation... | 244 |
| [the-shadow-self-intrinsic](skills/the-shadow-self-intrinsic/SKILL.md) | Detect and mitigate intrinsic value misalignment in LLM agent systems using the IMPRESS scenario-driven framework. Use w... | 234 |
| [timbre-aware-llm-based-direct-speech-to-speech](skills/timbre-aware-llm-based-direct-speech-to-speech/SKILL.md) | Build direct speech-to-speech translation systems that preserve speaker identity using LLM-based architectures with timb... | 210 |
| [tokenomics-quantifying-where-tokens](skills/tokenomics-quantifying-where-tokens/SKILL.md) | Analyze and optimize token consumption in LLM-based multi-agent software engineering workflows. Maps agent execution tra... | 227 |
| [toolself-unifying-task-execution](skills/toolself-unifying-task-execution/SKILL.md) | Implement self-reconfiguring agent workflows where configuration (sub-goals, strategy, toolbox, context) is a mutable to... | 217 |
| [toolweaver-weaving-collaborative-semantics](skills/toolweaver-weaving-collaborative-semantics/SKILL.md) | Design scalable tool retrieval systems using hierarchical code tokenization that captures collaborative tool semantics. ... | 181 |
| [topt-task-oriented-prompt-tuning](skills/topt-task-oriented-prompt-tuning/SKILL.md) | Apply Task-Oriented Prompt Tuning (ToPT) to build urban region representation learning pipelines that combine spatial-aw... | 184 |
| [toward-cognitive-supersensing-multimodal](skills/toward-cognitive-supersensing-multimodal/SKILL.md) | Apply Cognitive Supersensing to multimodal reasoning tasks -- augmenting text-only chain-of-thought with latent visual r... | 173 |
| [toward-culturally-aligned-ontology-guided](skills/toward-culturally-aligned-ontology-guided/SKILL.md) | Ontology-guided multi-agent reasoning for culturally aligned LLM outputs. Use when building systems that must respect cu... | 190 |
| [towards-ai-evaluation-domain-specific](skills/towards-ai-evaluation-domain-specific/SKILL.md) | Build and evaluate domain-specific RAG systems with iterative user-feedback refinement, source grounding, and structured... | 260 |
| [towards-holographic-characteristic-short-text](skills/towards-holographic-characteristic-short-text/SKILL.md) | Apply the Holographic Characteristic of LLMs to generate efficient short text by extracting keywords early then completi... | 150 |
| [towards-understanding-best-practices](skills/towards-understanding-best-practices/SKILL.md) | Quantize vision-language models (VLMs) component-by-component using optimal bit-width strategies derived from multimodal... | 189 |
| [tracemem-weaving-narrative-memory](skills/tracemem-weaving-narrative-memory/SKILL.md) | Build structured narrative memory systems from conversational traces using TraceMem's three-stage pipeline (segmentation... | 229 |
| [trailblazer-history-guided-reinforcement-learning](skills/trailblazer-history-guided-reinforcement-learning/SKILL.md) | Build history-aware RL pipelines for multi-turn LLM red-teaming and safety evaluation. Implements attention-weighted int... | 244 |
| [training-data-selection-gradient](skills/training-data-selection-gradient/SKILL.md) | Implement Orthogonal Gradient Selection (OGS) for efficient domain adaptation of LLMs—select training data whose gradien... | 196 |
| [training-multi-turn-search-agent](skills/training-multi-turn-search-agent/SKILL.md) | Build and train multi-turn search agents using BranPO (Branching Relative Policy Optimization) with contrastive dynamic ... | 177 |
| [trifuse-enhancing-attention-based-gui](skills/trifuse-enhancing-attention-based-gui/SKILL.md) | Implement training-free GUI grounding by fusing MLLM attention maps, OCR text cues, and icon caption semantics via Conse... | 162 |
| [trust-one-round-confidence](skills/trust-one-round-confidence/SKILL.md) | Estimate LLM output confidence from structural signals in a single pass -- no multi-sampling needed. Use when: 'check if... | 209 |
| [ts-debate-multimodal-collaborative-debate](skills/ts-debate-multimodal-collaborative-debate/SKILL.md) | Zero-shot time series reasoning via modality-specialized multi-agent debate. Assigns dedicated text, visual, and numeric... | 232 |
| [tsrbench-comprehensive-multi-task-multi-modal](skills/tsrbench-comprehensive-multi-task-multi-modal/SKILL.md) | Evaluate and build multi-modal time series reasoning pipelines using the TSRBench framework. Covers perception, reasonin... | 206 |
| [ttcs-test-time-curriculum-synthesis](skills/ttcs-test-time-curriculum-synthesis/SKILL.md) | Implement a co-evolving test-time curriculum synthesis framework where a question synthesizer and reasoning solver itera... | 226 |
| [tutorial-reasoning-ir-ir](skills/tutorial-reasoning-ir-ir/SKILL.md) | Build reasoning-enhanced information retrieval pipelines that go beyond semantic matching. Applies five methodological f... | 249 |
| [ui-venus-15-technical-report](skills/ui-venus-15-technical-report/SKILL.md) | Build GUI automation agents using UI-Venus-1.5 patterns: screenshot-only perception, coordinate-based grounding, traject... | 252 |
| [unikie-bench-benchmarking-multimodal-key](skills/unikie-bench-benchmarking-multimodal-key/SKILL.md) | Extract structured key information from document images using schema-guided prompting for LMMs. Builds KIE pipelines tha... | 292 |
| [unintended-memorization-sensitive-information](skills/unintended-memorization-sensitive-information/SKILL.md) | Audit fine-tuned LLMs for unintended PII memorization and apply privacy-preserving mitigations. Use when: 'audit my mode... | 199 |
| [unit-based-agent-semi-cascaded-full-duplex](skills/unit-based-agent-semi-cascaded-full-duplex/SKILL.md) | Build full-duplex voice dialogue systems using unit-based agent decomposition and semi-cascaded pipelines. Trigger phras... | 247 |
| [unveiling-cognitive-compass-theory-of-mind-guided](skills/unveiling-cognitive-compass-theory-of-mind-guided/SKILL.md) | Apply Theory-of-Mind (ToM) guided reasoning chains to multimodal emotion analysis tasks. Decomposes emotional reasoning ... | 197 |
| [urdubench-urdu-reasoning-benchmark](skills/urdubench-urdu-reasoning-benchmark/SKILL.md) | Build high-quality reasoning benchmarks for Urdu and other low-resource languages using contextually ensembled translati... | 171 |
| [use-graph-it-needs](skills/use-graph-it-needs/SKILL.md) | Implement adaptive RAG pipelines that route queries to dense retrieval, graph-based retrieval, or a weighted fusion base... | 254 |
| [valueflow-measuring-propagation-value](skills/valueflow-measuring-propagation-value/SKILL.md) | Measure and analyze how value perturbations propagate through multi-agent LLM systems using the ValueFlow framework. Dia... | 180 |
| [vectra-metric-dataset-visual](skills/vectra-metric-dataset-visual/SKILL.md) | Assess visual quality of translated product images using Vectra's 14-dimension scoring framework. Use when: 'evaluate tr... | 305 |
| [veq-modality-adaptive-quantization-moe](skills/veq-modality-adaptive-quantization-moe/SKILL.md) | Apply VEQ modality-adaptive quantization to compress MoE Vision-Language Models with minimal accuracy loss. Implements d... | 199 |
| [vespo-variational-sequence-level-soft](skills/vespo-variational-sequence-level-soft/SKILL.md) | Implement VESPO (Variational Sequence-Level Soft Policy Optimization) for stable off-policy LLM reinforcement learning. ... | 189 |
| [videothinker-building-agentic-videollms](skills/videothinker-building-agentic-videollms/SKILL.md) | Build agentic video understanding systems with LLM-guided tool reasoning. Implements the VideoThinker pattern: confidenc... | 204 |
| [vidvec-unlocking-video-mllm](skills/vidvec-unlocking-video-mllm/SKILL.md) | Extract high-quality video-text embeddings from generative MLLMs using intermediate-layer representations and text-only ... | 197 |
| [vihermes-graph-grounded-multihop-question](skills/vihermes-graph-grounded-multihop-question/SKILL.md) | Build graph-grounded multihop QA systems over regulatory and hierarchically structured documents. Combines vector simila... | 264 |
| [villain-at-averimatec-verifying](skills/villain-at-averimatec-verifying/SKILL.md) | Build multi-agent fact-checking pipelines that verify image-text claims through modality-specific analysis, cross-modal ... | 248 |
| [viola-video-in-context-learning](skills/viola-video-in-context-learning/SKILL.md) | Apply the VIOLA framework for label-efficient in-context learning on video or multimodal data. Uses density-uncertainty-... | 221 |
| [vision-deepresearch-benchmark-rethinking-visual-te](skills/vision-deepresearch-benchmark-rethinking-visual-te/SKILL.md) | Build and evaluate Vision-DeepResearch pipelines that combine cropped visual search with multi-hop textual search for ro... | 216 |
| [vision-representations-artificial-intelligence](skills/vision-representations-artificial-intelligence/SKILL.md) | Build autonomous driving safety systems using vision-language models (VLMs) for hazard detection, trajectory planning, a... | 281 |
| [visor-visual-spatial-object](skills/visor-visual-spatial-object/SKILL.md) | Implement VISOR-style three-stage visual spatial reasoning (think, think-summary, action) for embodied navigation and ob... | 198 |
| [vistira-closing-image-text-modality](skills/vistira-closing-image-text-modality/SKILL.md) | Solve math problems from images by decomposing them into interleaved natural-language rationales and executable Python c... | 189 |
| [visual-reasoning-over-time](skills/visual-reasoning-over-time/SKILL.md) | Analyze time series data using the MAS4TS Analyzer-Reasoner-Executor multi-agent paradigm: convert series to plots, extr... | 180 |
| [vividface-real-time-realistic-facial](skills/vividface-real-time-realistic-facial/SKILL.md) | Build real-time facial expression shadowing pipelines for humanoid robots using the VividFace two-module architecture (m... | 277 |
| [vlm-guided-iterative-refinement-surgical](skills/vlm-guided-iterative-refinement-surgical/SKILL.md) | Build iterative VLM-guided refinement pipelines for image segmentation tasks, especially surgical/medical imagery. Uses ... | 255 |
| [vowelprompt-hearing-speech-emotions](skills/vowelprompt-hearing-speech-emotions/SKILL.md) | Build speech emotion recognition pipelines that augment LLMs with vowel-level prosodic features converted to natural lan... | 287 |
| [voxmorph-scalable-zero-shot-voice](skills/voxmorph-scalable-zero-shot-voice/SKILL.md) | Build and deploy zero-shot voice identity morphing pipelines using disentangled prosody/timbre embeddings and Spherical ... | 203 |
| [vtc-r1-vision-text-compression-long-context](skills/vtc-r1-vision-text-compression-long-context/SKILL.md) | Implement VTC-R1 vision-text compression for efficient long-context reasoning. Renders intermediate reasoning segments i... | 269 |
| [what-makes-low-bit-quantization-aware](skills/what-makes-low-bit-quantization-aware/SKILL.md) | Implement the Reasoning-QAT pipeline for low-bit quantization-aware training of reasoning LLMs. Combines PTQ initializat... | 189 |
| [what-should-cite-rag](skills/what-should-cite-rag/SKILL.md) | Build multi-level RAG pipelines for academic citation prediction and literature discovery. Use when the user asks to 'fi... | 193 |
| [when-agents-misremember-collectively](skills/when-agents-misremember-collectively/SKILL.md) | Detect, measure, and defend against collective false-memory propagation (the Mandela Effect) in LLM multi-agent systems.... | 225 |
| [when-better-prompts-hurt](skills/when-better-prompts-hurt/SKILL.md) | Evaluation-driven prompt iteration using the Define-Test-Diagnose-Fix loop and Minimum Viable Evaluation Suite (MVES). P... | 196 |
| [when-evaluation-becomes-side](skills/when-evaluation-becomes-side/SKILL.md) | Detect and mitigate regime leakage in AI systems -- the information-theoretic vulnerability where models distinguish eva... | 237 |
| [when-iterative-rag-beats](skills/when-iterative-rag-beats/SKILL.md) | Build iterative retrieval-reasoning RAG pipelines that outperform single-shot retrieval, using staged evidence gathering... | 244 |
| [when-much-imagine-adaptive](skills/when-much-imagine-adaptive/SKILL.md) | Adaptive test-time scaling framework that decides WHEN and HOW MUCH to invoke expensive generative steps (world models, ... | 228 |
| [whispers-wealth-red-teaming-googles](skills/whispers-wealth-red-teaming-googles/SKILL.md) | Red-team LLM-based agentic payment systems against prompt injection attacks targeting transaction integrity and credenti... | 218 |
| [who-gets-which-message](skills/who-gets-which-message/SKILL.md) | Audit demographic bias in LLM-generated targeted text. Detects age- and gender-based stereotyping in personalized messag... | 221 |
| [why-reasoning-fails-plan](skills/why-reasoning-fails-plan/SKILL.md) | Apply FLARE (Future-aware Lookahead with Reward Estimation) to long-horizon coding tasks. Replaces greedy step-by-step r... | 218 |
| [wiki-live-challenge-challenging](skills/wiki-live-challenge-challenging/SKILL.md) | Evaluate deep research agents and LLM-generated long-form articles using the Wiki Live Challenge framework: 39 fine-grai... | 264 |
| [wildreward-learning-reward-in-the-wild](skills/wildreward-learning-reward-in-the-wild/SKILL.md) | Build reward models from implicit user feedback in chat logs using ordinal regression instead of annotated preference pa... | 175 |
| [xai-clip-roi-guided-perturbation-framework](skills/xai-clip-roi-guided-perturbation-framework/SKILL.md) | Build ROI-guided perturbation pipelines for explainable medical image segmentation using CLIP embeddings. Generates boun... | 226 |
| [xlist-hate-checklist-based-framework-interpretable](skills/xlist-hate-checklist-based-framework-interpretable/SKILL.md) | Decompose hate speech detection into a checklist of ten concept-level binary questions answered independently by an LLM,... | 229 |
| [your-secretly-contains-personality](skills/your-secretly-contains-personality/SKILL.md) | Extract and activate persona-specialized subnetworks from LLMs using activation-guided pruning and contrastive masking. ... | 233 |
| [zero-shot-product-attribute-labeling](skills/zero-shot-product-attribute-labeling/SKILL.md) | Extract and classify product attributes from images using Vision-Language Models with structured prompts and a three-tie... | 268 |
| [zero2text-zero-training-cross-domain-inversion](skills/zero2text-zero-training-cross-domain-inversion/SKILL.md) | Implement embedding inversion attacks that reconstruct original text from vector embeddings without training data, using... | 188 |

---

## Agentic Systems

**346 skills**

| Skill | Description | Lines |
|-------|-------------|-------|
| [3-secbench-large-scale-evaluation-suite-security](skills/3-secbench-large-scale-evaluation-suite-security/SKILL.md) | Evaluate and harden LLM-based autonomous agents against adversarial attacks using the α³-SecBench layered security frame... | 182 |
| [3d-space-as-scratchpad-editable](skills/3d-space-as-scratchpad-editable/SKILL.md) | Build agentic pipelines that use 3D scene layout as an intermediate reasoning workspace for controllable, spatially-accu... | 172 |
| [a-mapreduce-executing-wide-search](skills/a-mapreduce-executing-wide-search/SKILL.md) | Execute large-scale breadth-oriented search and retrieval tasks using the A-MapReduce pattern: decompose a wide query in... | 196 |
| [a-rag-scaling-agentic-retrieval-augmented](skills/a-rag-scaling-agentic-retrieval-augmented/SKILL.md) | > | 253 |
| [a2rag-adaptive-agentic-graph](skills/a2rag-adaptive-agentic-graph/SKILL.md) | Build adaptive, cost-aware Graph-RAG pipelines that route queries through escalating retrieval stages (local -> bridge -... | 229 |
| [accelerating-social-science-research](skills/accelerating-social-science-research/SKILL.md) | Implement the EXPERIGEN agentic framework for automated hypothesis generation and empirical validation on datasets. Uses... | 177 |
| [acegrpo-adaptive-curriculum-group](skills/acegrpo-adaptive-curriculum-group/SKILL.md) | Adaptive curriculum-driven iterative optimization for autonomous ML engineering tasks. Uses Evolving Data Buffers and Le... | 220 |
| [adaptive-confidence-gating-multi-agent](skills/adaptive-confidence-gating-multi-agent/SKILL.md) | Multi-agent code generation using structured debate with adaptive confidence gating. Three specialized agents (User/Prod... | 184 |
| [aegis-governance-integrity-security](skills/aegis-governance-integrity-security/SKILL.md) | Red-team and harden AI voice agents and LLM-powered service systems against adversarial misuse using the Aegis framework... | 237 |
| [aero-autonomous-evolutionary-reasoning](skills/aero-autonomous-evolutionary-reasoning/SKILL.md) | Apply the AERO dual-loop self-evolution framework to iteratively improve reasoning on complex tasks. Uses entropy-based ... | 160 |
| [affective-flow-emotional-support](skills/affective-flow-emotional-support/SKILL.md) | Build emotionally supportive multi-turn conversation systems using the AFlow framework — affective flow modeling with MC... | 216 |
| [agent-based-software-artifact-evaluation](skills/agent-based-software-artifact-evaluation/SKILL.md) | Automatically evaluate software research artifacts (code repositories with READMEs) by constructing dependency-aware com... | 203 |
| [agent-fence-mapping-security-vulnerabilities](skills/agent-fence-mapping-security-vulnerabilities/SKILL.md) | > | 211 |
| [agent-primitives-reusable-latent](skills/agent-primitives-reusable-latent/SKILL.md) | Design and orchestrate multi-agent systems using reusable Agent Primitives (Review, Voting/Selection, Planning/Execution... | 252 |
| [agent2agent-threats-safety-critical-assistants](skills/agent2agent-threats-safety-critical-assistants/SKILL.md) | Threat model multi-agent LLM systems using the AgentHeLLM framework -- formally separating asset identification from att... | 207 |
| [agentark-distilling-multi-agent-intelligence](skills/agentark-distilling-multi-agent-intelligence/SKILL.md) | Distill multi-agent debate reasoning into a single LLM's behavior. Apply AgentArk's three-tier distillation strategy to ... | 199 |
| [agentcgroup-understanding-controlling-os](skills/agentcgroup-understanding-controlling-os/SKILL.md) | Design and implement OS-level resource controls for sandboxed AI agents using hierarchical cgroups, eBPF enforcement, an... | 231 |
| [agentcpm-report-interleaving-drafting-deepening](skills/agentcpm-report-interleaving-drafting-deepening/SKILL.md) | > | 232 |
| [agentdog-diagnostic-guardrail-framework](skills/agentdog-diagnostic-guardrail-framework/SKILL.md) | > | 340 |
| [agentdrive-open-benchmark-dataset](skills/agentdrive-open-benchmark-dataset/SKILL.md) | Generate structured autonomous driving scenarios and MCQ benchmarks using AgentDrive's factorized 7-axis prompt-to-JSON ... | 284 |
| [agentic-ai-healthcare-medicine](skills/agentic-ai-healthcare-medicine/SKILL.md) | Design, evaluate, and improve LLM-based agentic systems for healthcare using a seven-dimensional taxonomy with 29 sub-di... | 274 |
| [agentic-reinforcement-learning-empowers](skills/agentic-reinforcement-learning-empowers/SKILL.md) | Build tool-augmented agent systems that decouple domain reasoning from knowledge storage, following the ChemCRAFT patter... | 242 |
| [agentic-very-long-video](skills/agentic-very-long-video/SKILL.md) | Build agentic systems for understanding very long video streams (hours to weeks) using entity scene graphs, multi-tool p... | 240 |
| [agenticpay-multi-agent-negotiation-system](skills/agenticpay-multi-agent-negotiation-system/SKILL.md) | Build multi-agent LLM negotiation systems where buyer and seller agents reach deals through natural language. Use when a... | 226 |
| [agenticscr-an-autonomous-agentic](skills/agenticscr-an-autonomous-agentic/SKILL.md) | > | 177 |
| [agenticsimlaw-juvenile-courtroom-multi-agent](skills/agenticsimlaw-juvenile-courtroom-multi-agent/SKILL.md) | Structured multi-agent courtroom debate for explainable high-stakes tabular decisions. Use when: 'set up a multi-agent d... | 181 |
| [agentskiller-scaling-generalist-agent](skills/agentskiller-scaling-generalist-agent/SKILL.md) | Synthesize multi-turn agent interaction data across semantically linked domains using DAG-based pipelines, domain ontolo... | 178 |
| [agentsm-semantic-memory-agentic](skills/agentsm-semantic-memory-agentic/SKILL.md) | Agentic Text-to-SQL with semantic memory that captures and reuses structured execution traces. Use when: 'write SQL for ... | 213 |
| [agentstepper-interactive-debugging-software](skills/agentstepper-interactive-debugging-software/SKILL.md) | Interactive debugging of LLM-powered software development agents using structured trajectory analysis, stepwise executio... | 197 |
| [agentsys-secure-dynamic-agents](skills/agentsys-secure-dynamic-agents/SKILL.md) | > | 191 |
| [agenttrace-structured-logging-framework](skills/agenttrace-structured-logging-framework/SKILL.md) | Implement structured, multi-surface observability logging for LLM agent systems using the AgentTrace pattern: operationa... | 152 |
| [agentxray-white-boxing-agentic-systems](skills/agentxray-white-boxing-agentic-systems/SKILL.md) | Reverse-engineer black-box agentic systems into editable, interpretable workflows using search-based reconstruction. Use... | 168 |
| [agyn-multi-agent-system-team-based](skills/agyn-multi-agent-system-team-based/SKILL.md) | Orchestrate multi-agent teams for autonomous software engineering using the Agyn methodology: coordinator, researcher, i... | 202 |
| [ai-agent-for-reverseengineering](skills/ai-agent-for-reverseengineering/SKILL.md) | > | 205 |
| [ai-agent-systems-supply](skills/ai-agent-systems-supply/SKILL.md) | > | 277 |
| [ai-my-values-user](skills/ai-my-values-user/SKILL.md) | Build value-aligned conversational agents using the VAPT (Value-Alignment Perception Toolkit) framework from CHI '26. Ex... | 236 |
| [aidev-studying-ai-coding](skills/aidev-studying-ai-coding/SKILL.md) | Analyze AI coding agent activity on GitHub repositories using the AIDev methodology. Identify agentic PRs, measure agent... | 195 |
| [alignagent-adaptive-learner-intelligence](skills/alignagent-adaptive-learner-intelligence/SKILL.md) | | | 319 |
| [alrm-agentic-robotic-manipulation](skills/alrm-agentic-robotic-manipulation/SKILL.md) | > | 131 |
| [ama-adaptive-memory-multi-agent](skills/ama-adaptive-memory-multi-agent/SKILL.md) | Build adaptive memory systems using coordinated multi-agent collaboration with hierarchical storage and consistency main... | 224 |
| [amem4rec-leveraging-cross-user-similarity](skills/amem4rec-leveraging-cross-user-similarity/SKILL.md) | Build agentic recommendation systems that learn collaborative filtering signals through cross-user memory evolution -- n... | 273 |
| [an-cost-efficient-agentic-framework](skills/an-cost-efficient-agentic-framework/SKILL.md) | Audit Ethereum smart contracts for business logic vulnerabilities using Heimdallr's four-phase agentic pipeline: functio... | 202 |
| [analyticsgpt-workflow-scientometric-question](skills/analyticsgpt-workflow-scientometric-question/SKILL.md) | Build sequential LLM pipelines for scientometric question answering over academic databases. Decomposes meta-scientific ... | 306 |
| [anonymization-enhanced-privacy-protection-mobile-g](skills/anonymization-enhanced-privacy-protection-mobile-g/SKILL.md) | Implement available-but-invisible privacy protection for mobile GUI agents using PII-aware anonymization with determinis... | 178 |
| [aorchestra-automating-sub-agent-creation](skills/aorchestra-automating-sub-agent-creation/SKILL.md) | Dynamically create specialized sub-agents for complex multi-step tasks using the AOrchestra pattern: decompose goals, th... | 203 |
| [apex-agents](skills/apex-agents/SKILL.md) | | | 223 |
| [asa-training-free-representation-engineering](skills/asa-training-free-representation-engineering/SKILL.md) | Implement Activation Steering Adapter (ASA) for training-free tool-calling improvement in LLM agents. Use when: 'fix laz... | 170 |
| [audiorouter-data-audio-understanding](skills/audiorouter-data-audio-understanding/SKILL.md) | Build audio understanding systems that route between internal LLM reasoning and external audio tools using a lightweight... | 249 |
| [audit-after-segmentation-reference-free](skills/audit-after-segmentation-reference-free/SKILL.md) | Build reference-free mask quality assessment pipelines for multimodal segmentation systems. Implements the MQ-Auditor pa... | 251 |
| [automated-multiple-mini-interview](skills/automated-multiple-mini-interview/SKILL.md) | Multi-agent framework for scoring subjective, open-ended responses (interviews, essays, reflections) using transcript re... | 189 |
| [automated-rubrics-reliable-evaluation](skills/automated-rubrics-reliable-evaluation/SKILL.md) | Generate fine-grained evaluation rubrics for medical dialogue systems using a retrieval-augmented multi-agent pipeline. ... | 180 |
| [automated-structural-testing-llm-based](skills/automated-structural-testing-llm-based/SKILL.md) | Write structural tests for LLM-based agents using trace-based assertions, mocked LLM responses, and the test automation ... | 242 |
| [automating-computational-reproducibility-social](skills/automating-computational-reproducibility-social/SKILL.md) | Diagnose and repair failing computational research code to restore reproducibility. Uses an agent-based iterative workfl... | 189 |
| [autonomous-business-system-neuro-symbolic](skills/autonomous-business-system-neuro-symbolic/SKILL.md) | | | 249 |
| [autonomous-chain-of-thought-distillation-graph-bas](skills/autonomous-chain-of-thought-distillation-graph-bas/SKILL.md) | Implement FraudCoT-style graph-aware chain-of-thought distillation for fraud detection on text-attributed graphs. Combin... | 199 |
| [autonomous-data-processing-meta-agents](skills/autonomous-data-processing-meta-agents/SKILL.md) | Build self-managing data processing pipelines using hierarchical meta-agent orchestration. Decomposes complex data tasks... | 216 |
| [autonomous-multi-agent-ai-high-throughput](skills/autonomous-multi-agent-ai-high-throughput/SKILL.md) | | | 189 |
| [avenir-web-human-experience-imitating-multimodal-w](skills/avenir-web-human-experience-imitating-multimodal-w/SKILL.md) | Build robust web automation agents using Mixture of Grounding Experts, experience-imitation planning, and task-tracking ... | 375 |
| [bayesflow-probability-inference-framework](skills/bayesflow-probability-inference-framework/SKILL.md) | Generate high-quality multi-step LLM workflows using Bayesian inference with parallel look-ahead rollouts and importance... | 204 |
| [behavioural-representational-evaluation-goal-direc](skills/behavioural-representational-evaluation-goal-direc/SKILL.md) | Evaluate goal-directedness of LLM agents by combining behavioural benchmarking against optimal policies with interpretab... | 172 |
| [benchmarking-reward-hack-detection](skills/benchmarking-reward-hack-detection/SKILL.md) | Detect reward hacking in AI-generated code trajectories using contrastive analysis from the TRACE benchmark. Use when: '... | 190 |
| [beyond-accuracy-cognitive-load](skills/beyond-accuracy-cognitive-load/SKILL.md) | Analyze and reduce cognitive load in tool-use agent workflows using the Cognitive Load Framework from AAAI 2026. Diagnos... | 218 |
| [birdturk-adaptation-bird-text-to-sql](skills/birdturk-adaptation-bird-text-to-sql/SKILL.md) | Adapt Text-to-SQL systems and benchmarks for non-English, morphologically rich languages using controlled translation pi... | 242 |
| [blind-gods-broken-screens](skills/blind-gods-broken-screens/SKILL.md) | Architect secure, intent-centric agent systems using the Aura pattern: Hub-and-Spoke agent topology, cryptographic ident... | 219 |
| [cam-causality-based-analysis-framework](skills/cam-causality-based-analysis-framework/SKILL.md) | Analyze and optimize multi-agent code generation pipelines using causality-based importance ranking of intermediate feat... | 153 |
| [can-implement-agent-based-odd-based](skills/can-implement-agent-based-odd-based/SKILL.md) | Translate ODD protocol specifications into validated, executable agent-based model (ABM) code in Python. Use when the us... | 208 |
| [can-truly-embody-human](skills/can-truly-embody-human/SKILL.md) | Evaluate and improve personality-behavior alignment in LLM simulations of human social interactions. Uses the BFI-IRP ev... | 206 |
| [ci4a-semantic-component-interfaces](skills/ci4a-semantic-component-interfaces/SKILL.md) | Build semantic component interfaces that expose UI components as structured tool primitives for AI agent automation. Use... | 272 |
| [closing-reasoning-gaps-clinical](skills/closing-reasoning-gaps-clinical/SKILL.md) | Build systems that detect and fix reasoning gaps in LLM agents by comparing their chain-of-thought against reference rea... | 196 |
| [co-redteam-orchestrated-security-discovery](skills/co-redteam-orchestrated-security-discovery/SKILL.md) | Multi-agent security vulnerability discovery and exploitation using Co-RedTeam's orchestrated workflow. Decomposes secur... | 197 |
| [cognitive-platform-engineering-autonomous](skills/cognitive-platform-engineering-autonomous/SKILL.md) | Build autonomous cloud operations using a four-plane cognitive architecture (Sensing, Reasoning, Orchestration, Experien... | 247 |
| [commcp-multi-agent-coordination-llm-based](skills/commcp-multi-agent-coordination-llm-based/SKILL.md) | Build decentralized multi-agent coordination systems using LLM-based communication calibrated with conformal prediction.... | 225 |
| [comparing-ai-coding-agents](skills/comparing-ai-coding-agents/SKILL.md) | > | 207 |
| [completing-missing-annotation-multi-agent](skills/completing-missing-annotation-multi-agent/SKILL.md) | Multi-agent debate framework for relevance assessment and annotation completion. Uses opposing-stance LLM agents with it... | 251 |
| [constrained-process-maps-multi-agent](skills/constrained-process-maps-multi-agent/SKILL.md) | Build multi-agent workflows structured as constrained DAG process maps with Monte Carlo uncertainty estimation. Each age... | 244 |
| [contextevolve-multi-agent-context-compression](skills/contextevolve-multi-agent-context-compression/SKILL.md) | Multi-agent iterative code optimization using context compression. Decomposes optimization into three agents (Summarizer... | 175 |
| [conversation-non-verifiable-learning-self-evolving](skills/conversation-non-verifiable-learning-self-evolving/SKILL.md) | | | 241 |
| [core-ubiquitous-6g-intelligence](skills/core-ubiquitous-6g-intelligence/SKILL.md) | Design and implement multi-LLM agent orchestration systems over hierarchical compute tiers using the CORE framework patt... | 195 |
| [corpusqa-10-million-token](skills/corpusqa-10-million-token/SKILL.md) | Corpus-level QA over massive document collections using memory-augmented agentic processing. Synthesize answers that req... | 179 |
| [covagent-overcoming-30-curse](skills/covagent-overcoming-30-curse/SKILL.md) | Boost Android app test coverage beyond the 30% activity ceiling using agentic static analysis of Smali code, component t... | 174 |
| [cowork-x-experience-optimized-co-evolution-multi-a](skills/cowork-x-experience-optimized-co-evolution-multi-a/SKILL.md) | Build multi-agent collaboration systems with experience-driven co-evolution using HTN skill libraries and post-episode o... | 150 |
| [creditaudit-2textnd-dimension-evaluation](skills/creditaudit-2textnd-dimension-evaluation/SKILL.md) | Evaluate and select LLMs using CreditAudit's 2D framework: mean ability plus stability risk (fluctuation) across system ... | 211 |
| [cua-skill-develop-skills-computer](skills/cua-skill-develop-skills-computer/SKILL.md) | Build reusable, parameterized skill libraries for computer-using agents (CUAs). Decomposes GUI automation into Skill Cel... | 297 |
| [curate-train-refine-closed-loop-agentic-framework-](skills/curate-train-refine-closed-loop-agentic-framework/SKILL.md) | Build lightweight text classifiers from zero labeled data using an agentic Curate-Train-Refine loop. An LLM generates sy... | 148 |
| [cve-factory-scaling-expert-level-agentic](skills/cve-factory-scaling-expert-level-agentic/SKILL.md) | > | 240 |
| [darwin-dynamic-agentically-rewriting](skills/darwin-dynamic-agentically-rewriting/SKILL.md) | Evolutionary multi-agent code optimization using genetic algorithms. Agents mutate each other's training/configuration c... | 167 |
| [data-centric-interpretability-llm-based-multi-agen](skills/data-centric-interpretability-llm-based-multi-agen/SKILL.md) | Analyze LLM agent behavior across training runs using Sparse Autoencoder (SAE) features and LLM-summarizer pipelines. Gr... | 189 |
| [datacross-unified-benchmark-agent](skills/datacross-unified-benchmark-agent/SKILL.md) | Cross-modal data analysis agent that unifies structured sources (SQL, CSV, JSON) with unstructured visual documents (sca... | 203 |
| [david-vs-goliath-verifiable](skills/david-vs-goliath-verifiable/SKILL.md) | Audit and harden tool-augmented AI agent systems against Tag-Along Attacks -- adversarial agent-to-agent jailbreaks that... | 165 |
| [davinci-dev-agent-native-mid-training-software](skills/davinci-dev-agent-native-mid-training-software/SKILL.md) | Apply daVinci-Dev's agent-native workflow to software engineering tasks: navigate repos, localize bugs, plan edits, appl... | 171 |
| [deep-search-hierarchical-meta-cognitive](skills/deep-search-hierarchical-meta-cognitive/SKILL.md) | Implement hierarchical meta-cognitive monitoring for deep search agents. Embeds a two-tier self-monitoring system (fast ... | 188 |
| [deepimagesearch-benchmarking-multimodal-agents](skills/deepimagesearch-benchmarking-multimodal-agents/SKILL.md) | Build agentic image retrieval systems that perform multi-step contextual reasoning over visual histories instead of isol... | 198 |
| [deepplanning-benchmarking-long-horizon-agentic](skills/deepplanning-benchmarking-long-horizon-agentic/SKILL.md) | Solve long-horizon planning tasks with verifiable constraints using the DeepPlanning methodology: proactive information ... | 155 |
| [devops-gym-benchmarking-ai-agents](skills/devops-gym-benchmarking-ai-agents/SKILL.md) | Apply the DevOps-Gym methodology to systematically tackle full-cycle DevOps tasks: build/configuration repair, runtime m... | 170 |
| [dllm-agent-see-farther](skills/dllm-agent-see-farther/SKILL.md) | Design and implement multi-agent workflows using the DeepDiver hierarchical orchestration pattern with diffusion-inspire... | 163 |
| [dllm-searcher-adapting-diffusion-large](skills/dllm-searcher-adapting-diffusion-large/SKILL.md) | Implement the P-ReAct parallel reasoning-and-acting agent paradigm from DLLM-Searcher, which overlaps tool execution wit... | 256 |
| [do-reasoning-ask-questions](skills/do-reasoning-ask-questions/SKILL.md) | Information-theoretic question-asking framework for disambiguating user intent through structured yes/no questions. Uses... | 183 |
| [dr-mas-stable-reinforcement-learning](skills/dr-mas-stable-reinforcement-learning/SKILL.md) | Design and implement stable reinforcement learning pipelines for multi-agent LLM systems using agent-wise advantage norm... | 203 |
| [dynamic-role-assignment-multi-agent](skills/dynamic-role-assignment-multi-agent/SKILL.md) | Dynamically assign specialized roles to multiple AI agents via a meta-debate protocol (proposal + peer review) before ru... | 182 |
| [dynaweb-model-based-reinforcement-learning](skills/dynaweb-model-based-reinforcement-learning/SKILL.md) | Build model-based RL training pipelines for web agents using learned world models (environment simulators) that predict ... | 193 |
| [dziribot-rag-intelligent-conversational](skills/dziribot-rag-intelligent-conversational/SKILL.md) | Build dialect-aware RAG conversational agents that handle non-standard orthography, code-switching, and multi-script inp... | 236 |
| [ecg-agent-on-device-tool-calling-agent](skills/ecg-agent-on-device-tool-calling-agent/SKILL.md) | Build on-device LLM tool-calling agents for multi-turn biomedical signal dialogue, following the ECG-Agent architecture.... | 296 |
| [effgen-enabling-small-language](skills/effgen-enabling-small-language/SKILL.md) | Deploy and optimize small language models (SLMs) as autonomous agents using the effGen framework. Implements prompt comp... | 193 |
| [eft-cot-multi-agent-chain-of-thought-framework](skills/eft-cot-multi-agent-chain-of-thought-framework/SKILL.md) | Build multi-agent emotion-focused therapy (EFT) reasoning pipelines for empathetic mental health Q&A systems. Uses a bot... | 300 |
| [ema-policy-gradient-taming](skills/ema-policy-gradient-taming/SKILL.md) | Implement EMA Policy Gradient (EMA-PG) for stabilizing reinforcement learning fine-tuning of LLMs. Combines an Exponenti... | 228 |
| [embocoach-bench-benchmarking-ai-agents](skills/embocoach-bench-benchmarking-ai-agents/SKILL.md) | | | 230 |
| [embodied-task-planning-graph-informed](skills/embodied-task-planning-graph-informed/SKILL.md) | Structure long-horizon task planning using graph-based memory and bounded lookahead. Use when asked to: 'plan a multi-st... | 179 |
| [emotion-llamav2-mmeverse-framework-benchmark](skills/emotion-llamav2-mmeverse-framework-benchmark/SKILL.md) | Build multimodal emotion understanding systems using the Emotion-LLaMAv2 architecture and MMEVerse benchmark methodology... | 232 |
| [empirical-mcts-continuous-agent-evolution](skills/empirical-mcts-continuous-agent-evolution/SKILL.md) | Applies Empirical-MCTS dual-loop reasoning: structured tree search with persistent memory that accumulates experience ac... | 195 |
| [entworld-holistic-environment-benchmark](skills/entworld-holistic-environment-benchmark/SKILL.md) | Build verifiable enterprise GUI agent benchmarks using schema-grounded task generation and SQL-based deterministic verif... | 158 |
| [epistemic-context-learning-building](skills/epistemic-context-learning-building/SKILL.md) | Build trust-aware multi-agent systems using Epistemic Context Learning (ECL). Constructs peer reliability profiles from ... | 210 |
| [es-memeval-benchmarking-conversational-agents](skills/es-memeval-benchmarking-conversational-agents/SKILL.md) | Build and evaluate long-term memory systems for conversational agents using the ES-MemEval five-capability framework (in... | 226 |
| [evermembench-benchmarking-long-term-interactive](skills/evermembench-benchmarking-long-term-interactive/SKILL.md) | Build and evaluate long-term conversational memory systems for multi-party, multi-topic dialogues. Implements the EverMe... | 182 |
| [evocodebench-human-performance-benchmark-self-evol](skills/evocodebench-human-performance-benchmark-self-evol/SKILL.md) | Self-evolving code generation with iterative reflection and revision. Applies a feedback-driven loop where code is submi... | 174 |
| [evoconfig-self-evolving-multi-agent-systems](skills/evoconfig-self-evolving-multi-agent-systems/SKILL.md) | Autonomous environment configuration using multi-agent diagnosis and self-evolving error repair. Use when: 'set up the d... | 194 |
| [evolving-tool-user-creator](skills/evolving-tool-user-creator/SKILL.md) | Transform Claude from a static tool user into a dynamic tool creator using the UCT (User-to-Creator Transformation) fram... | 181 |
| [experience-driven-multi-agent-systems-training-fre](skills/experience-driven-multi-agent-systems-training-fre/SKILL.md) | Build self-evolving multi-agent systems that accumulate tool-level expertise through structured interaction without mode... | 168 |
| [exploring-reasoning-reward-agents](skills/exploring-reasoning-reward-agents/SKILL.md) | | | 219 |
| [fademem-biologically-inspired-forgetting-agent](skills/fademem-biologically-inspired-forgetting-agent/SKILL.md) | > | 222 |
| [farm-field-aware-resolution-intelligent](skills/farm-field-aware-resolution-intelligent/SKILL.md) | Build intelligent trigger-action automation systems using FARM's two-stage architecture: contrastive retrieval + multi-a... | 187 |
| [fat-cat-document-driven-metacognitive-multi-agent](skills/fat-cat-document-driven-metacognitive-multi-agent/SKILL.md) | > | 225 |
| [featurebench-benchmarking-agentic-coding](skills/featurebench-benchmarking-agentic-coding/SKILL.md) | Extract feature-level coding tasks from repositories using test-driven dependency graph tracing. Use when the user says ... | 178 |
| [fimi-domain-specific-indian-finance](skills/fimi-domain-specific-indian-finance/SKILL.md) | Build domain-specialized AI agents for Indian financial systems (UPI, NPCI, RBI) using multi-stage training pipeline pat... | 243 |
| [flyaoc-evaluating-agentic-ontology](skills/flyaoc-evaluating-agentic-ontology/SKILL.md) | Build multi-agent systems for end-to-end ontology curation from scientific literature. Applies FlyAOC's agent architectu... | 184 |
| [forest-chat-adapting-vision-language-agents](skills/forest-chat-adapting-vision-language-agents/SKILL.md) | Build LLM-orchestrated agents for bi-temporal satellite image change analysis, combining vision-language models with too... | 362 |
| [from-assistant-double-agent](skills/from-assistant-double-agent/SKILL.md) | Security audit and hardening for personalized LLM-based agents against prompt injection, tool poisoning, and memory atta... | 230 |
| [from-assumptions-actions-turning](skills/from-assumptions-actions-turning/SKILL.md) | Build uncertainty-aware planners for multi-agent systems using the PCE (Planner-Composer-Evaluator) decision tree framew... | 242 |
| [from-features-actions-explainability](skills/from-features-actions-explainability/SKILL.md) | Diagnose and explain failures in agentic AI systems using trace-based rubric evaluation, bridging static feature attribu... | 207 |
| [from-helpfulness-toxic-proactivity](skills/from-helpfulness-toxic-proactivity/SKILL.md) | Diagnose and mitigate Toxic Proactivity in LLM agent systems -- the failure mode where agents override ethical constrain... | 194 |
| [from-passive-metric-active](skills/from-passive-metric-active/SKILL.md) | Build systems that use LLM uncertainty as an active control signal -- routing computation, triggering tool calls, enabli... | 268 |
| [from-perception-action-spatial](skills/from-perception-action-spatial/SKILL.md) | Design and implement spatially-aware AI agent systems using hierarchical memory, GNN-LLM integration, and world models. ... | 217 |
| [from-pragmas-partners-symbiotic](skills/from-pragmas-partners-symbiotic/SKILL.md) | Agentic High-Level Synthesis (HLS) optimization: autonomously analyze, insert, and tune C/C++ HLS pragmas (pipeline, unr... | 186 |
| [from-prompt-response-goal-directed-systems](skills/from-prompt-response-goal-directed-systems/SKILL.md) | Design production-grade agentic AI architectures with separated cognition/execution layers, typed tool interfaces, multi... | 177 |
| [from-task-solving-robust](skills/from-task-solving-robust/SKILL.md) | Build LLM agent workflows that stay robust under partial observability, noisy signals, shifting environments, and intern... | 199 |
| [fullstack-agent-enhancing-agentic-fullstack](skills/fullstack-agent-enhancing-agentic-fullstack/SKILL.md) | Build production-grade full-stack web applications using a three-agent pipeline (Planning, Backend, Frontend) with devel... | 146 |
| [gamedevbench-evaluating-agentic-capabilities](skills/gamedevbench-evaluating-agentic-capabilities/SKILL.md) | Agentic game development with visual feedback loops for Godot Engine projects. Applies the GameDevBench methodology: nav... | 187 |
| [gametalk-training-strategic-conversation](skills/gametalk-training-strategic-conversation/SKILL.md) | Build multi-agent strategic conversation systems where LLMs negotiate, coordinate, and optimize long-term objectives thr... | 207 |
| [graph-based-agent-memory-taxonomy](skills/graph-based-agent-memory-taxonomy/SKILL.md) | Design and implement graph-based memory systems for LLM agents following the extraction-storage-retrieval-evolution life... | 279 |
| [graphagents-knowledge-graph-guided-agentic](skills/graphagents-knowledge-graph-guided-agentic/SKILL.md) | Build multi-agent pipelines that use knowledge graphs to guide LLM reasoning across domains. Agents specialize in proble... | 185 |
| [graphdancer-training-explore-reason](skills/graphdancer-training-explore-reason/SKILL.md) | Build agentic graph-exploration systems where an LLM navigates heterogeneous knowledge graphs through interleaved reason... | 243 |
| [graphseek-next-generation-graph-analytics](skills/graphseek-next-generation-graph-analytics/SKILL.md) | Build LLM-powered graph analytics systems using the GraphSeek two-plane architecture: a Semantic Catalog for planning ov... | 151 |
| [haif-human-ai-integration-framework](skills/haif-human-ai-integration-framework/SKILL.md) | Apply the HAIF protocol to organize hybrid human-AI team workflows with tiered autonomy, delegation governance, and vali... | 206 |
| [hallucination-resistant-security-planning](skills/hallucination-resistant-security-planning/SKILL.md) | > | 259 |
| [hidden-licensing-risks-llmware](skills/hidden-licensing-risks-llmware/SKILL.md) | Detect license incompatibilities across LLM supply chains (OSS repos, models, datasets) using the LiAgent multi-agent ex... | 182 |
| [humans-welcome-observe-first-look](skills/humans-welcome-observe-first-look/SKILL.md) | Analyze AI agent social network activity using topic taxonomy classification and multi-level toxicity scoring. Detects c... | 191 |
| [ic-eo-interpretable-code-based-assistant](skills/ic-eo-interpretable-code-based-assistant/SKILL.md) | Build conversational Earth Observation agents that turn natural-language queries into executable, auditable Python workf... | 215 |
| [ide-bench-evaluating-as-ide](skills/ide-bench-evaluating-as-ide/SKILL.md) | Apply IDE-Bench's structured agent workflow for tackling real-world software engineering tasks: systematic exploration b... | 149 |
| [internalizing-multi-agent-reasoning-accurate](skills/internalizing-multi-agent-reasoning-accurate/SKILL.md) | Distill multi-agent reasoning into a single efficient model for recommendation or retrieval. Use when: 'build a recommen... | 174 |
| [internet-agentic-ai-incentive-compatible](skills/internet-agentic-ai-incentive-compatible/SKILL.md) | > | 208 |
| [interpreting-agentic-systems-beyond](skills/interpreting-agentic-systems-beyond/SKILL.md) | Audit and instrument agentic AI systems for system-level interpretability and accountability. Embeds traceability, causa... | 329 |
| [isd-agent-bench-comprehensive-benchmark-evaluating](skills/isd-agent-bench-comprehensive-benchmark-evaluating/SKILL.md) | Build and evaluate LLM-based Instructional Design agents using the ADDIE framework, Context Matrix scenario generation, ... | 210 |
| [iterative-refinement-improves-compositional](skills/iterative-refinement-improves-compositional/SKILL.md) | Implement iterative critic-guided refinement loops for compositional image generation. Uses a VLM critic to progressivel... | 199 |
| [jade-bridging-strategic-operational-gap](skills/jade-bridging-strategic-operational-gap/SKILL.md) | Build jointly-optimized agentic RAG pipelines using the JADE pattern: a central planner co-adapted with specialized exec... | 248 |
| [jaf-judge-agent-forest](skills/jaf-judge-agent-forest/SKILL.md) | | | 213 |
| [just-in-time-reinforcement-learning-continual](skills/just-in-time-reinforcement-learning-continual/SKILL.md) | Implement JitRL-style continual learning for LLM agents: training-free policy optimization via experience memory, advant... | 203 |
| [large-geolocation-extraction-humanitarian](skills/large-geolocation-extraction-humanitarian/SKILL.md) | Extract and geocode location mentions from humanitarian and crisis texts using a two-step LLM pipeline: few-shot NER for... | 213 |
| [latent-chain-of-thought-as-planning](skills/latent-chain-of-thought-as-planning/SKILL.md) | Decouple reasoning from verbalization using PLaT-inspired latent planning. Maintains a broad solution space through para... | 200 |
| [latentchem-textual-cot-latent](skills/latentchem-textual-cot-latent/SKILL.md) | Apply LatentChem's latent-space reasoning paradigm to chemical computation tasks -- replacing verbose textual Chain-of-T... | 189 |
| [learning-compose-cross-domain-agentic](skills/learning-compose-cross-domain-agentic/SKILL.md) | Generate cross-domain agentic workflows using decompose-recompose-decide composition over reusable capability bases. Use... | 159 |
| [learning-decentralized-collaboration-multi-agent](skills/learning-decentralized-collaboration-multi-agent/SKILL.md) | Design and orchestrate decentralized multi-LLM collaboration systems using Multi-Agent Actor-Critic (MAAC) patterns from... | 218 |
| [learning-irrecoverable-error-localized-policy](skills/learning-irrecoverable-error-localized-policy/SKILL.md) | Debug multi-step tool-using agent pipelines by localizing the first irrecoverable error via binary-search rollback, then... | 175 |
| [legalmalr-multi-agent-query-understanding](skills/legalmalr-multi-agent-query-understanding/SKILL.md) | Multi-agent query reformulation and LLM reranking for retrieval over legal, regulatory, or domain-specific corpora. Use ... | 168 |
| [legalone-family-foundation-reliable](skills/legalone-family-foundation-reliable/SKILL.md) | Build domain-specialized LLM training pipelines using the LegalOne three-phase methodology: Plasticity-Adjusted Sampling... | 259 |
| [lemon-agent-technical-report](skills/lemon-agent-technical-report/SKILL.md) | Orchestrate multi-agent workflows using the Lemon Agent orchestrator-worker pattern with hierarchical scheduling, progre... | 186 |
| [lhaw-controllable-underspecification-long-horizon](skills/lhaw-controllable-underspecification-long-horizon/SKILL.md) | Detect and handle ambiguity in long-horizon agent tasks using the LHAW framework. Systematically identify underspecified... | 170 |
| [linguistagent-a-reflective-multimodel](skills/linguistagent-a-reflective-multimodel/SKILL.md) | > | 241 |
| [lingxidiagbench-multi-agent-framework-benchmarking](skills/lingxidiagbench-multi-agent-framework-benchmarking/SKILL.md) | Build multi-agent benchmarking systems with role-separated agents (simulator, interviewer, evaluator) for structured mul... | 216 |
| [live-evo-online-evolution-agentic](skills/live-evo-online-evolution-agentic/SKILL.md) | Implement online self-evolving memory for LLM agents using dual-bank architecture (Experience Bank + Meta-Guideline Bank... | 204 |
| [livemedbench-contamination-free-medical-benchmark](skills/livemedbench-contamination-free-medical-benchmark/SKILL.md) | Build contamination-free LLM evaluation pipelines with multi-agent data curation and automated rubric-based scoring. Use... | 296 |
| [livibench-omnimodal-benchmark-interactive](skills/livibench-omnimodal-benchmark-interactive/SKILL.md) | Build omnimodal benchmarks and evaluation pipelines for interactive video understanding (livestreams, real-time comments... | 238 |
| [llm-autodp-automatic-data-processing](skills/llm-autodp-automatic-data-processing/SKILL.md) | Automatically generate and optimize data processing pipelines for LLM fine-tuning datasets using an agent-driven iterati... | 168 |
| [llm-enhanced-reinforcement-learning-long-term](skills/llm-enhanced-reinforcement-learning-long-term/SKILL.md) | Build hierarchical recommendation systems that combine LLM semantic planning with RL fine-grained optimization for long-... | 247 |
| [llm-in-sandbox-elicits-general-agentic](skills/llm-in-sandbox-elicits-general-agentic/SKILL.md) | Solve non-code tasks (math, science, long-context, formatting) by treating the terminal as a sandbox for exploration: wr... | 198 |
| [loca-bench-benchmarking-agents-under](skills/loca-bench-benchmarking-agents-under/SKILL.md) | Apply context management strategies from LOCA-bench to prevent context rot in long-running agent tasks. Implements progr... | 169 |
| [localv-exploiting-information-locality](skills/localv-exploiting-information-locality/SKILL.md) | Multi-agent framework for generating large-scale Verilog/RTL code from long hardware specifications by decomposing long-... | 138 |
| [locomo-plus-beyond-factual-cognitive-memory](skills/locomo-plus-beyond-factual-cognitive-memory/SKILL.md) | Build and evaluate cognitive memory systems for LLM dialogue agents that retain implicit user constraints (state, goals,... | 216 |
| [longcat-flash-thinking-2601-technical-report](skills/longcat-flash-thinking-2601-technical-report/SKILL.md) | Build robust multi-tool agentic pipelines with noise-aware execution, parallel reasoning, and environment scaling patter... | 311 |
| [made-benchmark-environments-closed-loop](skills/made-benchmark-environments-closed-loop/SKILL.md) | Build closed-loop discovery benchmarks where an agent iteratively proposes, evaluates, and refines candidates under a fi... | 144 |
| [magellan-autonomous-discovery-compiler](skills/magellan-autonomous-discovery-compiler/SKILL.md) | Evolve compiler optimization heuristics by coupling LLM code generation with evolutionary search and autotuning. Synthes... | 158 |
| [malicious-agent-skills-wild](skills/malicious-agent-skills-wild/SKILL.md) | > | 264 |
| [marble-multi-agent-reasoning-bioinformatics](skills/marble-multi-agent-reasoning-bioinformatics/SKILL.md) | Iteratively refine bioinformatics and ML models using MARBLE's multi-agent debate framework with role-specialized agents... | 206 |
| [marti-mars2-scaling-multi-agent-self-search-reinfo](skills/marti-mars2-scaling-multi-agent-self-search-reinfo/SKILL.md) | Multi-agent tree-search code generation using heterogeneous agent collaboration with error-feedback refinement. Spawns m... | 204 |
| [mas-prove-understanding-process-verification](skills/mas-prove-understanding-process-verification/SKILL.md) | Design and implement process verification for multi-agent LLM systems. Add intermediate-step evaluation to multi-agent w... | 237 |
| [mascot-multi-agent-socio-collaborative-companion](skills/mascot-multi-agent-socio-collaborative-companion/SKILL.md) | Design and orchestrate multi-agent companion systems where each agent maintains a distinct persona and contributes diver... | 244 |
| [mata-multiagent-framework-for](skills/mata-multiagent-framework-for/SKILL.md) | Multi-agent table question answering using MATA's three-path reasoning strategy (Chain-of-Thought, Program-of-Thought, T... | 167 |
| [mata-trainable-hierarchical-automaton](skills/mata-trainable-hierarchical-automaton/SKILL.md) | Build multi-agent visual reasoning systems using hierarchical finite-state automata with a trainable hyper agent that or... | 303 |
| [mathliblemma-folklore-lemma-generation](skills/mathliblemma-folklore-lemma-generation/SKILL.md) | Multi-agent system for discovering and formalizing missing 'folklore' lemmas in Lean 4 / Mathlib. Identifies gaps in for... | 181 |
| [mcp-atlas-large-scale-benchmark-tool-use](skills/mcp-atlas-large-scale-benchmark-tool-use/SKILL.md) | Design and evaluate multi-server MCP tool-use benchmarks using claims-based scoring rubrics. Use when: 'benchmark my MCP... | 210 |
| [medbeads-agent-native-immutable-data](skills/medbeads-agent-native-immutable-data/SKILL.md) | Build immutable, agent-native medical data pipelines using Merkle DAG structures (MedBeads pattern). Converts mutable EM... | 182 |
| [medsam-agent-empowering-interactive-medical](skills/medsam-agent-empowering-interactive-medical/SKILL.md) | | | 223 |
| [memadapter-fast-alignment-across](skills/memadapter-fast-alignment-across/SKILL.md) | Unify heterogeneous agent memory systems (explicit graphs, parametric weights, latent KV-caches) via generative subgraph... | 166 |
| [menvagent-scalable-polyglot-environment](skills/menvagent-scalable-polyglot-environment/SKILL.md) | Automated Docker environment construction for polyglot repositories using a Planning-Execution-Verification multi-agent ... | 188 |
| [mermaid-memory-enhanced-retrieval-reasoning](skills/mermaid-memory-enhanced-retrieval-reasoning/SKILL.md) | Memory-enhanced multi-agent retrieval and reasoning for veracity assessment and fact-checking. Use when: 'verify this cl... | 189 |
| [metagen-self-evolving-roles-topologies](skills/metagen-self-evolving-roles-topologies/SKILL.md) | | | 189 |
| [mirror-multi-agent-framework-iterative](skills/mirror-multi-agent-framework-iterative/SKILL.md) | Translate natural language optimization problems into mathematical models and solver code using MIRROR's multi-agent pip... | 166 |
| [mitigating-conversational-inertia-multi-turn](skills/mitigating-conversational-inertia-multi-turn/SKILL.md) | Detect and break conversational inertia in multi-turn agent interactions — where an LLM repeats its own prior actions as... | 236 |
| [moco-one-stop-shop-collaboration](skills/moco-one-stop-shop-collaboration/SKILL.md) | Design and implement multi-LM collaboration pipelines using the MoCo framework's 26 methods across four collaboration le... | 225 |
| [multi-agent-causal-reasoning-system](skills/multi-agent-causal-reasoning-system/SKILL.md) | Build multi-agent systems that discover causal rules from event sequences using specialized agents (causal discovery, co... | 225 |
| [multi-agent-collaborative-intrusion-detection](skills/multi-agent-collaborative-intrusion-detection/SKILL.md) | Build multi-agent intrusion detection systems using LLM-enhanced collaborative agents for network traffic classification... | 309 |
| [multi-agent-constraint-factorization-reveals](skills/multi-agent-constraint-factorization-reveals/SKILL.md) | Orchestrate multi-agent LLM pipelines using constraint factorization -- decomposing complex requirements into separate c... | 157 |
| [multi-agent-end-to-end-vulnerability-management](skills/multi-agent-end-to-end-vulnerability-management/SKILL.md) | Detect, confirm, repair, and validate recurring software vulnerabilities using a multi-agent pipeline modeled on MAVM. B... | 196 |
| [multi-agent-teams-hold-experts](skills/multi-agent-teams-hold-experts/SKILL.md) | Prevent expertise dilution in multi-agent LLM workflows by applying findings from 'Multi-Agent Teams Hold Experts Back' ... | 151 |
| [multi-agentic-ai-fairness-aware-accelerated](skills/multi-agentic-ai-fairness-aware-accelerated/SKILL.md) | Design and implement multi-agent systems for fairness-aware, low-latency inference orchestration across distributed edge... | 200 |
| [multi-field-tool-retrieval](skills/multi-field-tool-retrieval/SKILL.md) | Implement multi-field tool retrieval systems that decompose tool documentation into structured fields (description, para... | 284 |
| [multimodal-multi-agent-ransomware-analysis](skills/multimodal-multi-agent-ransomware-analysis/SKILL.md) | Build multimodal multi-agent pipelines for ransomware classification using specialized per-modality agents, autoencoder ... | 261 |
| [multivis-agent-multi-agent-framework-logic](skills/multivis-agent-multi-agent-framework-logic/SKILL.md) | Build reliable multi-agent data visualization pipelines with logic rule constraints. Use when: 'generate a chart from th... | 193 |
| [mulvul-retrieval-augmented-multi-agent-code](skills/mulvul-retrieval-augmented-multi-agent-code/SKILL.md) | Multi-agent vulnerability detection using coarse-to-fine routing, contrastive retrieval, and cross-model prompt evolutio... | 204 |
| [naamse-framework-evolutionary-security](skills/naamse-framework-evolutionary-security/SKILL.md) | Implement evolutionary security evaluation for AI agents using the NAAMSE framework — genetic prompt mutation, hierarchi... | 201 |
| [next-gen-captchas-leveraging-cognitive](skills/next-gen-captchas-leveraging-cognitive/SKILL.md) | Design and implement AI-resistant CAPTCHA systems that exploit the cognitive gap between humans and GUI agents. Covers p... | 253 |
| [note2chat-improving-multi-turn-clinical](skills/note2chat-improving-multi-turn-clinical/SKILL.md) | Build structured multi-turn clinical history-taking agents and diagnostic chatbots using the Note2Chat framework: conver... | 175 |
| [odysseyarena-benchmarking-long-horizon-active](skills/odysseyarena-benchmarking-long-horizon-active/SKILL.md) | Design and run inductive agent benchmarks where LLMs must discover hidden rules through long-horizon interaction loops r... | 176 |
| [omg-agent-robust-missing-modality](skills/omg-agent-robust-missing-modality/SKILL.md) | > | 280 |
| [omnicode-benchmark-evaluating-software](skills/omnicode-benchmark-evaluating-software/SKILL.md) | Evaluate and improve code across four software engineering dimensions: bug fixing, test generation, code review fixing, ... | 199 |
| [omnirag-agent-agentic-omnimodal-reasoning](skills/omnirag-agent-agentic-omnimodal-reasoning/SKILL.md) | Build agentic multimodal RAG pipelines that answer questions over long audio-video content under resource constraints. U... | 278 |
| [on-impact-agentsmd-files](skills/on-impact-agentsmd-files/SKILL.md) | Generate and optimize AGENTS.md / CLAUDE.md repository instruction files to reduce AI coding agent runtime and token con... | 196 |
| [on-uncertainty-model-based-multi-agent](skills/on-uncertainty-model-based-multi-agent/SKILL.md) | Apply entropy-based uncertainty analysis to multi-agent LLM systems. Diagnose when multi-agent setups hurt performance, ... | 184 |
| [ontology-to-tools-compilation-executable-semantic-](skills/ontology-to-tools-compilation-executable-semantic/SKILL.md) | Compile domain ontologies (OWL/RDFS/JSON-LD schemas) into executable tool interfaces with embedded semantic constraints,... | 192 |
| [openguandan-large-scale-imperfect-information](skills/openguandan-large-scale-imperfect-information/SKILL.md) | Build AI agents for the OpenGuanDan imperfect-information card game benchmark. Covers WebSocket client implementation, g... | 366 |
| [optimizing-small-sample-experience-learning-llm-ba](skills/optimizing-small-sample-experience-learning-llm-ba/SKILL.md) | Implement the ExperienceWeaver hierarchical experience-learning framework to improve text quality from small feedback se... | 195 |
| [outrunning-cutoffs-live-kernel](skills/outrunning-cutoffs-live-kernel/SKILL.md) | Build live-evolving kernel crash resolution benchmarks and agent environments using the Live-kBench/kEnv methodology. Se... | 153 |
| [pabu-progress-aware-belief-update](skills/pabu-progress-aware-belief-update/SKILL.md) | Apply Progress-Aware Belief Update (PABU) to build efficient LLM agents that track task progress and selectively retain ... | 210 |
| [pamas-self-adaptive-multi-agent-system](skills/pamas-self-adaptive-multi-agent-system/SKILL.md) | Build hierarchical multi-agent systems that detect misinformation, anomalies, and deceptive content using perspective-aw... | 162 |
| [paperbanana-automating-academic-illustration](skills/paperbanana-automating-academic-illustration/SKILL.md) | Generate publication-ready academic illustrations using a multi-agent pipeline inspired by PaperBanana. Orchestrates ret... | 197 |
| [papersearchqa-learning-search-reason](skills/papersearchqa-learning-search-reason/SKILL.md) | Build iterative search-and-reason agents for scientific literature QA. Uses the PaperSearchQA pattern: interleaved think... | 233 |
| [patch-to-poc-systematic-study-agentic](skills/patch-to-poc-systematic-study-agentic/SKILL.md) | > | 172 |
| [pathwise-planning-world-automated](skills/pathwise-planning-world-automated/SKILL.md) | Multi-agent heuristic design framework that uses an entailment graph, policy/world-model/critic agents, and routed refle... | 165 |
| [pearl-plan-exploration-adaptive](skills/pearl-plan-exploration-adaptive/SKILL.md) | Apply PEARL's two-phase tool orchestration: offline tool exploration to learn valid usage patterns and failure modes, th... | 172 |
| [peerrank-autonomous-evaluation-web-grounded](skills/peerrank-autonomous-evaluation-web-grounded/SKILL.md) | Implement PeerRank-style autonomous multi-model evaluation pipelines where LLMs symmetrically generate tasks, answer wit... | 215 |
| [perfguard-performance-aware-agent-visual](skills/perfguard-performance-aware-agent-visual/SKILL.md) | > | 227 |
| [planner-auditor-twin-agentic-discharge](skills/planner-auditor-twin-agentic-discharge/SKILL.md) | Implement a Planner-Auditor twin architecture that decouples LLM generation from deterministic validation with self-impr... | 189 |
| [polarmem-training-free-polarized-latent](skills/polarmem-training-free-polarized-latent/SKILL.md) | Build polarized memory systems for multimodal agents that encode both positive and negative evidence as graph constraint... | 183 |
| [prism-principled-framework-multi-agent](skills/prism-principled-framework-multi-agent/SKILL.md) | > | 138 |
| [proact-agentic-lookahead-interactive](skills/proact-agentic-lookahead-interactive/SKILL.md) | > | 241 |
| [prograph-r1-progress-aware-reinforcement-learning](skills/prograph-r1-progress-aware-reinforcement-learning/SKILL.md) | Build progress-aware GraphRAG agents that traverse knowledge graphs with structure-aware hypergraph retrieval and dense ... | 167 |
| [prompt-injection-attacks-agentic](skills/prompt-injection-attacks-agentic/SKILL.md) | > | 207 |
| [proxywar-dynamic-assessment-of](skills/proxywar-dynamic-assessment-of/SKILL.md) | Build competitive game-arena evaluation frameworks for LLM-generated code using ProxyWar's multi-layer pipeline: agent g... | 197 |
| [puda-private-user-dataset](skills/puda-private-user-dataset/SKILL.md) | Build privacy-preserving personalized AI systems using Puda's multi-granularity user data architecture. Implements clien... | 213 |
| [pull-requests-as-training](skills/pull-requests-as-training/SKILL.md) | Apply the Clean-PR agentless repo-level code editing protocol: decompose issues into file localization, fine-grained nav... | 199 |
| [qrs-rule-synthesizing-neuro-symbolic-triad](skills/qrs-rule-synthesizing-neuro-symbolic-triad/SKILL.md) | Autonomous vulnerability discovery using the QRS (Query, Review, Sanitize) neuro-symbolic triad. Generates CodeQL querie... | 229 |
| [quasar-universal-autonomous-system](skills/quasar-universal-autonomous-system/SKILL.md) | Build autonomous multi-scale scientific simulation pipelines using the QUASAR architecture: a Strategist-Operator-Evalua... | 165 |
| [query-efficient-agentic-graph-extraction](skills/query-efficient-agentic-graph-extraction/SKILL.md) | > | 239 |
| [rank-and-reason-multi-agent-collaboration-accelera](skills/rank-and-reason-multi-agent-collaboration-accelera/SKILL.md) | | | 245 |
| [rapid-real-time-deterministic-trajectory](skills/rapid-real-time-deterministic-trajectory/SKILL.md) | Distill diffusion-based trajectory planners into fast deterministic policies using score-regularized optimization and sa... | 185 |
| [rc-grpo-reward-conditioned-group-relative](skills/rc-grpo-reward-conditioned-group-relative/SKILL.md) | Implement reward-conditioned training pipelines for multi-turn tool-calling agents using RC-GRPO. Injects discrete rewar... | 228 |
| [reasoning-tool-use-compete-agentic](skills/reasoning-tool-use-compete-agentic/SKILL.md) | Diagnose and fix interference between reasoning and tool-use in agentic AI systems using LEAS attribution and DART-style... | 204 |
| [redvisor-reasoning-aware-prompt-injection](skills/redvisor-reasoning-aware-prompt-injection/SKILL.md) | Defend LLM applications against prompt injection using RedVisor's two-phase reasoning-then-responding architecture. Impl... | 223 |
| [refer-agent-collaborative-multi-agent-system](skills/refer-agent-collaborative-multi-agent-system/SKILL.md) | Build collaborative multi-agent systems that use alternating reasoning-reflection cycles with specialized agent roles, c... | 179 |
| [refuge-feature-generation-prediction](skills/refuge-feature-generation-prediction/SKILL.md) | Automated feature engineering for prediction tasks on relational databases using a multi-agent LLM pipeline. Generates, ... | 168 |
| [reprompt-prompt-generation-intelligent](skills/reprompt-prompt-generation-intelligent/SKILL.md) | Generate optimized system and user prompts for coding agents using requirements engineering principles from the REprompt... | 191 |
| [resagent-entropy-based-prior-point](skills/resagent-entropy-based-prior-point/SKILL.md) | Implement entropy-guided coarse-to-fine visual grounding pipelines for referring expression segmentation and point-promp... | 266 |
| [rethinking-role-entropy-optimizing](skills/rethinking-role-entropy-optimizing/SKILL.md) | Optimize LLM agent tool-use behavior using entropy reduction as a quality signal. Reduces excessive tool calls and impro... | 175 |
| [rethinking-value-agent-generated-tests](skills/rethinking-value-agent-generated-tests/SKILL.md) | Optimize agent test-writing strategy for issue resolution by reallocating interaction budget from excessive test generat... | 163 |
| [revisiting-salient-object-detection](skills/revisiting-salient-object-detection/SKILL.md) | Build observer-centric salient object detection systems using the Perceive-Reflect-Adjust agentic loop. Combines a Visio... | 257 |
| [robustexplain-evaluating-robustness-llm-based](skills/robustexplain-evaluating-robustness-llm-based/SKILL.md) | Evaluate robustness of LLM-generated recommendation explanations under realistic user behavior noise. Use when: 'test ex... | 211 |
| [roma-recursive-open-meta-agent](skills/roma-recursive-open-meta-agent/SKILL.md) | Decompose long-horizon, multi-step tasks using ROMA's recursive meta-agent pattern: Atomizer decides if a task needs spl... | 185 |
| [rulesmith-multi-agent-automated-game](skills/rulesmith-multi-agent-automated-game/SKILL.md) | Automated game balancing using multi-agent LLM self-play coupled with Bayesian optimization. Use when the user asks to '... | 193 |
| [safepred-predictive-guardrail-computer-using](skills/safepred-predictive-guardrail-computer-using/SKILL.md) | Implement predictive safety guardrails for computer-using agents and automated pipelines using world-model-based risk pr... | 179 |
| [scidatacopilot-agentic-data-preparation](skills/scidatacopilot-agentic-data-preparation/SKILL.md) | Build agentic pipelines that ingest heterogeneous raw scientific data, parse research intent, and produce analysis-ready... | 240 |
| [self-evolving-recommendation-system-end-to-end](skills/self-evolving-recommendation-system-end-to-end/SKILL.md) | Build autonomous ML optimization pipelines that use LLM agents to generate, evaluate, and deploy model improvements in a... | 168 |
| [semanticalli-caching-reasoning-not](skills/semanticalli-caching-reasoning-not/SKILL.md) | Implement pipeline-aware intermediate representation (IR) caching for agentic systems. Instead of caching final LLM resp... | 202 |
| [sera-soft-verified-repository-agents](skills/sera-soft-verified-repository-agents/SKILL.md) | > | 220 |
| [shardmemo-masked-moe-routing](skills/shardmemo-masked-moe-routing/SKILL.md) | Implement ShardMemo-style tiered, sharded memory with masked Mixture-of-Experts routing for agentic LLM systems. Use whe... | 176 |
| [shopsimulator-evaluating-exploring-rl-driven](skills/shopsimulator-evaluating-exploring-rl-driven/SKILL.md) | Build and evaluate LLM-based shopping assistant agents using structured multi-turn dialogue, personalized product search... | 285 |
| [sifting-noise-comparative-study](skills/sifting-noise-comparative-study/SKILL.md) | Filter false positives from static analysis security tools (SAST) using LLM-agent-driven triage. Applies iterative code ... | 154 |
| [skillrl-evolving-agents-recursive](skills/skillrl-evolving-agents-recursive/SKILL.md) | Build self-improving agent systems that distill raw execution traces into a hierarchical skill library (SkillBank) and r... | 184 |
| [smartoracle-agentic-approach](skills/smartoracle-agentic-approach/SKILL.md) | Agentic differential oracle for triaging cross-implementation discrepancies. Decomposes bug triage into specialized sub-... | 177 |
| [snapmla-efficient-longcontext-mla](skills/snapmla-efficient-longcontext-mla/SKILL.md) | While FP8 attention has shown substantial promise in innovations like FlashAttention-3, its integration into the decodin... | 88 |
| [socialveil-probing-social-intelligence](skills/socialveil-probing-social-intelligence/SKILL.md) | Stress-test LLM agents' social intelligence by injecting realistic communication barriers (semantic vagueness, sociocult... | 203 |
| [socratic-geo-synthetic-data-generation](skills/socratic-geo-synthetic-data-generation/SKILL.md) | Generate high-quality synthetic training data through multi-agent feedback loops where a Teacher agent creates parameter... | 226 |
| [solagent-specialized-multi-agent-framework](skills/solagent-specialized-multi-agent-framework/SKILL.md) | Generate secure, functionally correct Solidity smart contracts using a dual-loop refinement process: an inner loop that ... | 193 |
| [sparc-rag-adaptive-sequential-parallel-scaling](skills/sparc-rag-adaptive-sequential-parallel-scaling/SKILL.md) | Implement multi-agent RAG systems with coordinated sequential-parallel scaling and shared context management for complex... | 248 |
| [spectral-guardrails-agents-wild](skills/spectral-guardrails-agents-wild/SKILL.md) | Implement training-free hallucination detection for LLM agent tool calls using spectral analysis of attention topology. ... | 250 |
| [spider-sense-intrinsic-risk-sensing](skills/spider-sense-intrinsic-risk-sensing/SKILL.md) | Implement event-driven, hierarchical security screening for LLM agent systems using Intrinsic Risk Sensing. Adds latent ... | 212 |
| [spotagent-grounding-visual-geo-localization](skills/spotagent-grounding-visual-geo-localization/SKILL.md) | Build agentic geo-localization systems that combine vision-language model reasoning with tool-assisted verification usin... | 248 |
| [sqlagent-learning-explore-before](skills/sqlagent-learning-explore-before/SKILL.md) | Explore unfamiliar databases before writing SQL by building a local knowledge base of schema fragments, executable queri... | 201 |
| [st-raptor-agentic-system-semi-structured](skills/st-raptor-agentic-system-semi-structured/SKILL.md) | Agentic system for answering questions about semi-structured tables using tree-based structural modeling and multi-step ... | 222 |
| [star-similarity-guided-teacher-assisted-refinement](skills/star-similarity-guided-teacher-assisted-refinement/SKILL.md) | Distill function-calling capabilities from large language models into super-tiny models (0.6B-4B) using the STAR framewo... | 281 |
| [stateless-yet-not-forgetful](skills/stateless-yet-not-forgetful/SKILL.md) | Detect, audit, and defend against implicit memory channels in LLM-powered systems where models encode hidden state in ou... | 245 |
| [status-hierarchies](skills/status-hierarchies/SKILL.md) | Detect and mitigate status hierarchy bias in multi-agent LLM systems. Applies expectation states theory to audit deferen... | 229 |
| [step-35-flash-open](skills/step-35-flash-open/SKILL.md) | Build efficient agentic AI systems using sparse MoE routing, hybrid sliding-window/full attention, multi-token predictio... | 226 |
| [stepshield-not-whether-intervene](skills/stepshield-not-whether-intervene/SKILL.md) | Implement temporal safety monitoring for AI agent trajectories using StepShield's cascaded HybridGuard pattern. Detects ... | 282 |
| [strong-reasoning-isnt-enough](skills/strong-reasoning-isnt-enough/SKILL.md) | Build interactive diagnostic agents that systematically elicit evidence before concluding, using the REFINE (Reasoning-E... | 251 |
| [supchain-bench-benchmarking-real-world-supply](skills/supchain-bench-benchmarking-real-world-supply/SKILL.md) | Build reliable long-horizon supply chain agents using the SupChain-ReAct pattern: multi-path ReAct trajectories with maj... | 198 |
| [supporting-software-engineering-tasks](skills/supporting-software-engineering-tasks/SKILL.md) | Generate test scenarios from requirements and retrieve/analyze software engineering documents using a supervisor-worker ... | 169 |
| [swe-bench-mobile-agents-develop](skills/swe-bench-mobile-agents-develop/SKILL.md) | > | 247 |
| [swe-replay-test-time-scaling-software](skills/swe-replay-test-time-scaling-software/SKILL.md) | Efficient test-time scaling for software engineering agents using trajectory recycling and explore-exploit branching (SW... | 158 |
| [sycoeval-em-sycophancy-evaluation-simulated](skills/sycoeval-em-sycophancy-evaluation-simulated/SKILL.md) | Build multi-agent adversarial simulations to evaluate LLM sycophancy and policy compliance under social pressure. Use wh... | 241 |
| [symphony-synergistic-multi-agent-planning](skills/symphony-synergistic-multi-agent-planning/SKILL.md) | > | 201 |
| [synthagent-multi-agent-framework-realistic](skills/synthagent-multi-agent-framework-realistic/SKILL.md) | Build multi-agent pipelines that generate realistic synthetic patient profiles by integrating epidemiological data, medi... | 298 |
| [table-as-search-formulate-long-horizon-agentic](skills/table-as-search-formulate-long-horizon-agentic/SKILL.md) | Structured table-completion framework for long-horizon information seeking. Converts complex research queries into datab... | 203 |
| [task-oriented-robot-human-handovers-legged](skills/task-oriented-robot-human-handovers-legged/SKILL.md) | Implement task-oriented robot-to-human object handover systems using LLM-driven affordance reasoning and exemplar-based ... | 261 |
| [temp-r1-unified-autonomous-agent](skills/temp-r1-unified-autonomous-agent/SKILL.md) | Build autonomous agents that answer complex temporal questions over knowledge graphs or time-stamped datasets using stru... | 200 |
| [termigen-high-fidelity-environment-robust](skills/termigen-high-fidelity-environment-robust/SKILL.md) | Synthesize verifiable Docker-based task environments and error-resilient terminal agent trajectories using TermiGen's mu... | 181 |
| [the-clef-2026-finmmeval-lab](skills/the-clef-2026-finmmeval-lab/SKILL.md) | Build multilingual, multimodal financial AI evaluation pipelines using the FinMMEval framework. Covers financial exam QA... | 246 |
| [the-landscape-prompt-injection](skills/the-landscape-prompt-injection/SKILL.md) | Harden LLM agent systems against prompt injection using layered text/model/execution defenses and the AgentPI evaluation... | 244 |
| [the-necessity-unified-framework](skills/the-necessity-unified-framework/SKILL.md) | Design and implement standardized, reproducible evaluation harnesses for LLM-based agents. Eliminates confounding factor... | 185 |
| [the-shadow-self-intrinsic](skills/the-shadow-self-intrinsic/SKILL.md) | Detect and mitigate intrinsic value misalignment in LLM agent systems using the IMPRESS scenario-driven framework. Use w... | 234 |
| [thinking-frames-visual-context](skills/thinking-frames-visual-context/SKILL.md) | Decompose complex visual reasoning and spatial planning tasks into frame-by-frame intermediate steps, using visual conte... | 238 |
| [thinking-makes-agents-introverted](skills/thinking-makes-agents-introverted/SKILL.md) | > | 244 |
| [thinktank-me-multi-expert-framework-middle](skills/thinktank-me-multi-expert-framework-middle/SKILL.md) | Build multi-expert forecasting systems where specialized LLM agents collaborate through routing and aggregation to predi... | 207 |
| [timely-machine-awareness-time](skills/timely-machine-awareness-time/SKILL.md) | Apply time-budget-aware reasoning to agentic tasks with tool calls. Dynamically adjust strategy depth, tool call frequen... | 171 |
| [timemachine-bench-benchmark-evaluating-capabilitie](skills/timemachine-bench-benchmark-evaluating-capabilitie/SKILL.md) | Systematic dependency migration for Python projects. Diagnose and fix test failures caused by dependency updates using a... | 143 |
| [tokenomics-quantifying-where-tokens](skills/tokenomics-quantifying-where-tokens/SKILL.md) | Analyze and optimize token consumption in LLM-based multi-agent software engineering workflows. Maps agent execution tra... | 227 |
| [toolself-unifying-task-execution](skills/toolself-unifying-task-execution/SKILL.md) | Implement self-reconfiguring agent workflows where configuration (sub-goals, strategy, toolbox, context) is a mutable to... | 217 |
| [toward-cognitive-supersensing-multimodal](skills/toward-cognitive-supersensing-multimodal/SKILL.md) | Apply Cognitive Supersensing to multimodal reasoning tasks -- augmenting text-only chain-of-thought with latent visual r... | 173 |
| [toward-culturally-aligned-ontology-guided](skills/toward-culturally-aligned-ontology-guided/SKILL.md) | Ontology-guided multi-agent reasoning for culturally aligned LLM outputs. Use when building systems that must respect cu... | 190 |
| [towards-adaptive-scalable-robust](skills/towards-adaptive-scalable-robust/SKILL.md) | Implement RAPS (Reputation-Aware Publish-Subscribe) multi-agent coordination using intent-based pub/sub messaging, react... | 205 |
| [towards-agentic-intelligence-for](skills/towards-agentic-intelligence-for/SKILL.md) | | | 226 |
| [towards-automated-kernel-generation](skills/towards-automated-kernel-generation/SKILL.md) | Automate GPU kernel generation and optimization using LLM-driven agentic workflows with profiling feedback loops. Use wh... | 159 |
| [towards-autonomous-mathematics-research](skills/towards-autonomous-mathematics-research/SKILL.md) | Iterative generate-verify-revise agent for mathematical research problems. Implements the Aletheia loop: decompose a har... | 239 |
| [towards-declarative-agentic-layer](skills/towards-declarative-agentic-layer/SKILL.md) | Build grounded, declarative agentic architectures using the DALIA pattern: capability descriptors, discovery protocols, ... | 218 |
| [tracecoder-trace-driven-multi-agent-framework](skills/tracecoder-trace-driven-multi-agent-framework/SKILL.md) | Trace-driven debugging framework for LLM-generated code. Uses diagnostic probe instrumentation, causal trace analysis, a... | 191 |
| [tracemem-weaving-narrative-memory](skills/tracemem-weaving-narrative-memory/SKILL.md) | Build structured narrative memory systems from conversational traces using TraceMem's three-stage pipeline (segmentation... | 229 |
| [training-multi-turn-search-agent](skills/training-multi-turn-search-agent/SKILL.md) | Build and train multi-turn search agents using BranPO (Branching Relative Policy Optimization) with contrastive dynamic ... | 177 |
| [ts-debate-multimodal-collaborative-debate](skills/ts-debate-multimodal-collaborative-debate/SKILL.md) | Zero-shot time series reasoning via modality-specialized multi-agent debate. Assigns dedicated text, visual, and numeric... | 232 |
| [ui-venus-15-technical-report](skills/ui-venus-15-technical-report/SKILL.md) | Build GUI automation agents using UI-Venus-1.5 patterns: screenshot-only perception, coordinate-based grounding, traject... | 252 |
| [understanding-agent-scaling-llm-based](skills/understanding-agent-scaling-llm-based/SKILL.md) | Design diversity-aware multi-agent systems that maximize performance with fewer agents. Uses information-theoretic K* ef... | 204 |
| [unit-based-agent-semi-cascaded-full-duplex](skills/unit-based-agent-semi-cascaded-full-duplex/SKILL.md) | Build full-duplex voice dialogue systems using unit-based agent decomposition and semi-cascaded pipelines. Trigger phras... | 247 |
| [valueflow-measuring-propagation-value](skills/valueflow-measuring-propagation-value/SKILL.md) | Measure and analyze how value perturbations propagate through multi-agent LLM systems using the ValueFlow framework. Dia... | 180 |
| [veri-sure-contract-aware-multi-agent-framework](skills/veri-sure-contract-aware-multi-agent-framework/SKILL.md) | Generate functionally correct RTL/Verilog code using a contract-aware multi-agent workflow with formal verification. Tri... | 186 |
| [videothinker-building-agentic-videollms](skills/videothinker-building-agentic-videollms/SKILL.md) | Build agentic video understanding systems with LLM-guided tool reasoning. Implements the VideoThinker pattern: confidenc... | 204 |
| [villain-at-averimatec-verifying](skills/villain-at-averimatec-verifying/SKILL.md) | Build multi-agent fact-checking pipelines that verify image-text claims through modality-specific analysis, cross-modal ... | 248 |
| [vision-deepresearch-benchmark-rethinking-visual-te](skills/vision-deepresearch-benchmark-rethinking-visual-te/SKILL.md) | Build and evaluate Vision-DeepResearch pipelines that combine cropped visual search with multi-hop textual search for ro... | 216 |
| [vision-deepresearch-incentivizing-deepresearch-cap](skills/vision-deepresearch-incentivizing-deepresearch-cap/SKILL.md) | Multi-turn, multi-entity, multi-scale visual and textual deep research agent for answering complex questions about image... | 180 |
| [vision-representations-artificial-intelligence](skills/vision-representations-artificial-intelligence/SKILL.md) | Build autonomous driving safety systems using vision-language models (VLMs) for hazard detection, trajectory planning, a... | 281 |
| [visor-visual-spatial-object](skills/visor-visual-spatial-object/SKILL.md) | Implement VISOR-style three-stage visual spatial reasoning (think, think-summary, action) for embodied navigation and ob... | 198 |
| [visual-cognitive-demands-model-powered](skills/visual-cognitive-demands-model-powered/SKILL.md) | Evaluate visual and cognitive demands of in-vehicle LLM interfaces using the Monk et al. (2026) dual-metric framework. I... | 290 |
| [visual-reasoning-over-time](skills/visual-reasoning-over-time/SKILL.md) | Analyze time series data using the MAS4TS Analyzer-Reasoner-Executor multi-agent paradigm: convert series to plots, extr... | 180 |
| [vlm-guided-iterative-refinement-surgical](skills/vlm-guided-iterative-refinement-surgical/SKILL.md) | Build iterative VLM-guided refinement pipelines for image segmentation tasks, especially surgical/medical imagery. Uses ... | 255 |
| [vln-pilot-vision-language-as-autonomous](skills/vln-pilot-vision-language-as-autonomous/SKILL.md) | Build VLLM-driven autonomous navigation agents that interpret natural language instructions and ground them in visual ob... | 235 |
| [wdscaling-parallel-tool-calling-deep](skills/wdscaling-parallel-tool-calling-deep/SKILL.md) | Scale deep research tasks by issuing parallel tool calls (width) alongside sequential reasoning (depth), following the W... | 161 |
| [when-agents-fail-act](skills/when-agents-fail-act/SKILL.md) | > | 194 |
| [when-agents-fail-comprehensive](skills/when-agents-fail-comprehensive/SKILL.md) | Diagnose and fix bugs in LLM agent systems using a research-backed taxonomy of 11 bug types, 9 root causes, and 12 obser... | 240 |
| [when-agents-misremember-collectively](skills/when-agents-misremember-collectively/SKILL.md) | Detect, measure, and defend against collective false-memory propagation (the Mandela Effect) in LLM multi-agent systems.... | 225 |
| [when-better-prompts-hurt](skills/when-better-prompts-hurt/SKILL.md) | Evaluation-driven prompt iteration using the Define-Test-Diagnose-Fix loop and Minimum Viable Evaluation Suite (MVES). P... | 196 |
| [when-evaluation-becomes-side](skills/when-evaluation-becomes-side/SKILL.md) | Detect and mitigate regime leakage in AI systems -- the information-theoretic vulnerability where models distinguish eva... | 237 |
| [when-much-imagine-adaptive](skills/when-much-imagine-adaptive/SKILL.md) | Adaptive test-time scaling framework that decides WHEN and HOW MUCH to invoke expensive generative steps (world models, ... | 228 |
| [where-ai-coding-agents](skills/where-ai-coding-agents/SKILL.md) | Pre-flight checker that prevents AI coding agent PRs from failing, based on empirical analysis of 33k agent-authored PRs... | 193 |
| [whispers-wealth-red-teaming-googles](skills/whispers-wealth-red-teaming-googles/SKILL.md) | Red-team LLM-based agentic payment systems against prompt injection attacks targeting transaction integrity and credenti... | 218 |
| [who-deserves-reward-sharp](skills/who-deserves-reward-sharp/SKILL.md) | Apply SHARP (Shapley-based credit attribution) to design and optimize multi-agent systems where each agent's individual ... | 220 |
| [why-ai-agents-systematically](skills/why-ai-agents-systematically/SKILL.md) | > | 309 |
| [why-deep-research-agent](skills/why-deep-research-agent/SKILL.md) | > | 199 |
| [why-reasoning-fails-plan](skills/why-reasoning-fails-plan/SKILL.md) | Apply FLARE (Future-aware Lookahead with Reward Estimation) to long-horizon coding tasks. Replaces greedy step-by-step r... | 218 |
| [wideseek-r1-exploring-width-scaling](skills/wideseek-r1-exploring-width-scaling/SKILL.md) | Decompose broad information-seeking tasks into parallel subtasks using a lead-agent-subagent pattern with isolated conte... | 197 |
| [wiki-live-challenge-challenging](skills/wiki-live-challenge-challenging/SKILL.md) | Evaluate deep research agents and LLM-generated long-form articles using the Wiki Live Challenge framework: 39 fine-grai... | 264 |
| [yunque-deepresearch-technical-report](skills/yunque-deepresearch-technical-report/SKILL.md) | Hierarchical multi-agent deep research framework with dynamic context management and supervisor-based error recovery. Im... | 191 |

---

## Evaluation & Benchmarking

**263 skills**

| Skill | Description | Lines |
|-------|-------------|-------|
| [3-secbench-large-scale-evaluation-suite-security](skills/3-secbench-large-scale-evaluation-suite-security/SKILL.md) | Evaluate and harden LLM-based autonomous agents against adversarial attacks using the α³-SecBench layered security frame... | 182 |
| [aacr-bench-evaluating-automatic-code](skills/aacr-bench-evaluating-automatic-code/SKILL.md) | Perform repository-level automated code review on pull requests using hierarchical context retrieval and structured defe... | 187 |
| [aegis-governance-integrity-security](skills/aegis-governance-integrity-security/SKILL.md) | Red-team and harden AI voice agents and LLM-powered service systems against adversarial misuse using the Aegis framework... | 237 |
| [agent-based-software-artifact-evaluation](skills/agent-based-software-artifact-evaluation/SKILL.md) | Automatically evaluate software research artifacts (code repositories with READMEs) by constructing dependency-aware com... | 203 |
| [agentdrive-open-benchmark-dataset](skills/agentdrive-open-benchmark-dataset/SKILL.md) | Generate structured autonomous driving scenarios and MCQ benchmarks using AgentDrive's factorized 7-axis prompt-to-JSON ... | 284 |
| [agentic-ai-healthcare-medicine](skills/agentic-ai-healthcare-medicine/SKILL.md) | Design, evaluate, and improve LLM-based agentic systems for healthcare using a seven-dimensional taxonomy with 29 sub-di... | 274 |
| [aicd-bench-challenging-benchmark](skills/aicd-bench-challenging-benchmark/SKILL.md) | Detect whether source code was written by a human or generated by an AI model, attribute code to specific model families... | 211 |
| [aidev-studying-ai-coding](skills/aidev-studying-ai-coding/SKILL.md) | Analyze AI coding agent activity on GitHub repositories using the AIDev methodology. Identify agentic PRs, measure agent... | 195 |
| [analyticsgpt-workflow-scientometric-question](skills/analyticsgpt-workflow-scientometric-question/SKILL.md) | Build sequential LLM pipelines for scientometric question answering over academic databases. Decomposes meta-scientific ... | 306 |
| [aqascore-evaluating-semantic-alignment](skills/aqascore-evaluating-semantic-alignment/SKILL.md) | > | 210 |
| [are-open-weight-ready-social](skills/are-open-weight-ready-social/SKILL.md) | Build LLM-based content moderation pipelines using zero-shot classification with open-weight models. Implements the stru... | 264 |
| [arkeval-benchmarking-evaluating-automated](skills/arkeval-benchmarking-evaluating-automated/SKILL.md) | Automated ArkTS code repair using retrieval-augmented generation, LLM-based test oracle synthesis, and structured benchm... | 196 |
| [assessing-business-process-modeling](skills/assessing-business-process-modeling/SKILL.md) | Evaluate and generate BPMN process models from natural language using the BEF4LLM framework. Assess BPMN XML quality acr... | 229 |
| [assessing-domain-level-susceptibility-emergent](skills/assessing-domain-level-susceptibility-emergent/SKILL.md) | | | 287 |
| [assessing-quality-mental-health](skills/assessing-quality-mental-health/SKILL.md) | > | 374 |
| [assessment-generative-named-entity](skills/assessment-generative-named-entity/SKILL.md) | Build generative NER systems using LLMs with optimal output formats and prompt engineering. Use when: 'extract entities ... | 245 |
| [audit-after-segmentation-reference-free](skills/audit-after-segmentation-reference-free/SKILL.md) | Build reference-free mask quality assessment pipelines for multimodal segmentation systems. Implements the MQ-Auditor pa... | 251 |
| [automated-multiple-mini-interview](skills/automated-multiple-mini-interview/SKILL.md) | Multi-agent framework for scoring subjective, open-ended responses (interviews, essays, reflections) using transcript re... | 189 |
| [automated-rubrics-reliable-evaluation](skills/automated-rubrics-reliable-evaluation/SKILL.md) | Generate fine-grained evaluation rubrics for medical dialogue systems using a retrieval-augmented multi-agent pipeline. ... | 180 |
| [autonomous-chain-of-thought-distillation-graph-bas](skills/autonomous-chain-of-thought-distillation-graph-bas/SKILL.md) | Implement FraudCoT-style graph-aware chain-of-thought distillation for fraud detection on text-attributed graphs. Combin... | 199 |
| [avere-improving-audiovisual-emotion](skills/avere-improving-audiovisual-emotion/SKILL.md) | Build emotion-aware multimodal AI systems that resist spurious cue-emotion associations and hallucinated audiovisual evi... | 177 |
| [bass-benchmarking-audio-lms](skills/bass-benchmarking-audio-lms/SKILL.md) | Build evaluation benchmarks for audio language models using the BASS methodology — structured task taxonomies across str... | 260 |
| [behavioural-representational-evaluation-goal-direc](skills/behavioural-representational-evaluation-goal-direc/SKILL.md) | Evaluate goal-directedness of LLM agents by combining behavioural benchmarking against optimal policies with interpretab... | 172 |
| [benchmarking-abap-code-generation](skills/benchmarking-abap-code-generation/SKILL.md) | Generate syntactically correct and functional ABAP code using iterative compiler feedback loops. Applies the empirical m... | 183 |
| [benchmarking-pairwise-causal-discovery](skills/benchmarking-pairwise-causal-discovery/SKILL.md) | > | 204 |
| [benchmarking-reward-hack-detection](skills/benchmarking-reward-hack-detection/SKILL.md) | Detect reward hacking in AI-generated code trajectories using contrastive analysis from the TRACE benchmark. Use when: '... | 190 |
| [benchmarking-text-to-python-against-text-to-sql](skills/benchmarking-text-to-python-against-text-to-sql/SKILL.md) | Generate correct Python/Pandas code from natural language questions over tabular data, applying the Logic Completion Fra... | 188 |
| [benchmarking-uncertainty-calibration-long-form](skills/benchmarking-uncertainty-calibration-long-form/SKILL.md) | | | 210 |
| [benchmarking-zero-shot-few-shot-phishing](skills/benchmarking-zero-shot-few-shot-phishing/SKILL.md) | Detect phishing URLs using LLM zero-shot and few-shot prompting with structured classification prompts. Use when: 'class... | 215 |
| [better-generalizing-unseen-concepts](skills/better-generalizing-unseen-concepts/SKILL.md) | Build biomedical concept recognition systems that generalize to unseen ontology concepts using hierarchical indexing and... | 143 |
| [beyond-alignment-expanding-reasoning](skills/beyond-alignment-expanding-reasoning/SKILL.md) | Apply Manifold-Reshaping Policy Optimization (MRPO) to expand LLM reasoning capacity beyond alignment. Implements spectr... | 250 |
| [beyond-confidence-rhythms-reasoning](skills/beyond-confidence-rhythms-reasoning/SKILL.md) | Analyze and improve LLM prompt robustness using the Token Constraint Bound (delta-TCB) metric from the paper 'Beyond Con... | 169 |
| [beyond-holistic-scores-automatic](skills/beyond-holistic-scores-automatic/SKILL.md) | Build trait-based essay scoring systems that evaluate argumentative writing across multiple rubric dimensions (Content, ... | 201 |
| [beyond-needles-illusion-decoupled](skills/beyond-needles-illusion-decoupled/SKILL.md) | Decouple evidence access from evidence use when evaluating or building long-context and RAG systems under semantic inter... | 191 |
| [beyond-single-perspective-text](skills/beyond-single-perspective-text/SKILL.md) | Detect text anomalies (spam, phishing, harmful content) using multi-view embeddings from diverse language models, combin... | 195 |
| [beyond-speedup-utilizing](skills/beyond-speedup-utilizing/SKILL.md) | Reuse LLM KV caches as free embeddings for confidence scoring and adaptive fast/slow reasoning. Use when: 'extract embed... | 247 |
| [beyond-superficial-unlearning-sharpness-aware](skills/beyond-superficial-unlearning-sharpness-aware/SKILL.md) | Implement sharpness-aware robust erasure (SARE) for hallucination removal in multimodal LLMs. Uses Targeted-SAM to flatt... | 269 |
| [bias-ear-listener-assessing](skills/bias-ear-listener-assessing/SKILL.md) | Assess and audit bias in audio/speech language models using the BiasInEar framework. Evaluate multimodal LLM robustness ... | 182 |
| [biasscope-automated-detection-bias](skills/biasscope-automated-detection-bias/SKILL.md) | Automatically discover and test for hidden biases in LLM-as-a-Judge evaluation pipelines using the BiasScope framework. ... | 206 |
| [birdturk-adaptation-bird-text-to-sql](skills/birdturk-adaptation-bird-text-to-sql/SKILL.md) | Adapt Text-to-SQL systems and benchmarks for non-English, morphologically rich languages using controlled translation pi... | 242 |
| [bridging-academia-industry-comprehensive](skills/bridging-academia-industry-comprehensive/SKILL.md) | Attributed graph clustering pipelines using PyAGC's Encode-Cluster-Optimize framework. Triggers: 'cluster nodes in a gra... | 264 |
| [bridging-arithmetic-gap-cognitive](skills/bridging-arithmetic-gap-cognitive/SKILL.md) | Iterative Dual-Phase Financial-PoT: decouple semantic reasoning from arithmetic computation to eliminate calculation err... | 222 |
| [c3box-clip-based-class-incremental-learning](skills/c3box-clip-based-class-incremental-learning/SKILL.md) | Set up, configure, and run CLIP-based class-incremental learning experiments using the C3Box toolbox. Supports 17 CIL al... | 215 |
| [can-reasoning-be-trusted](skills/can-reasoning-be-trusted/SKILL.md) | Validate and score LLM-generated statistical reasoning using a three-axis rubric (Correctness 40%, Explanation 35%, Reas... | 203 |
| [can-small-handle-context-summarized](skills/can-small-handle-context-summarized/SKILL.md) | Build context-summarized multi-turn QA systems that let small language models (SLMs) handle customer-service dialogues w... | 254 |
| [can-truly-embody-human](skills/can-truly-embody-human/SKILL.md) | Evaluate and improve personality-behavior alignment in LLM simulations of human social interactions. Uses the BFI-IRP ev... | 206 |
| [can-we-classify-flaky](skills/can-we-classify-flaky/SKILL.md) | Analyze test suites for flaky tests using LLM-based classification with context-augmented reasoning. Applies findings fr... | 182 |
| [capture-flags-family-based-evaluation](skills/capture-flags-family-based-evaluation/SKILL.md) | > | 165 |
| [chipbench-next-step-benchmark-evaluating](skills/chipbench-next-step-benchmark-evaluating/SKILL.md) | Evaluate and improve LLM-generated hardware designs using ChipBench methodology: structured Verilog generation with hier... | 225 |
| [chunking-retrieval-re-ranking-empirical-evaluation](skills/chunking-retrieval-re-ranking-empirical-evaluation/SKILL.md) | Build and optimize two-stage RAG pipelines with bi-encoder retrieval, cross-encoder re-ranking, and empirically-validate... | 230 |
| [codecircuit-inferring-llm-generated-code](skills/codecircuit-inferring-llm-generated-code/SKILL.md) | Assess LLM-generated code correctness using attribution graph analysis inspired by mechanistic interpretability. Apply s... | 195 |
| [compar-ia-french-governments](skills/compar-ia-french-governments/SKILL.md) | Build multilingual LLM evaluation arenas and preference data collection pipelines modeled on France's compar:IA platform... | 310 |
| [compass-contrastive-learning-automated](skills/compass-contrastive-learning-automated/SKILL.md) | Assess patch correctness using contrastive learning on code representations. Applies semantic-preserving code transforma... | 180 |
| [completing-missing-annotation-multi-agent](skills/completing-missing-annotation-multi-agent/SKILL.md) | Multi-agent debate framework for relevance assessment and annotation completion. Uses opposing-stance LLM agents with it... | 251 |
| [comprehensive-evaluation-software-engineering](skills/comprehensive-evaluation-software-engineering/SKILL.md) | > | 285 |
| [computational-approach-visual-metonymy](skills/computational-approach-visual-metonymy/SKILL.md) | Generate and evaluate visual metonymy -- indirect visual representations that evoke concepts through associated cues rat... | 179 |
| [conceptual-cultural-index-metric](skills/conceptual-cultural-index-metric/SKILL.md) | > | 246 |
| [consistency-meets-verification-enhancing](skills/consistency-meets-verification-enhancing/SKILL.md) | Generate high-reliability test suites without ground-truth implementations using the ConVerTest pipeline: Self-Consisten... | 178 |
| [core-comprehensive-ontological-relation](skills/core-comprehensive-ontological-relation/SKILL.md) | Detect and prevent semantic collapse in LLM outputs — where models fabricate spurious relationships between unrelated co... | 216 |
| [corefine-confidence-guided-self-refinement-adaptiv](skills/corefine-confidence-guided-self-refinement-adaptiv/SKILL.md) | Confidence-guided self-refinement for adaptive reasoning. Implements the CoRefine pattern: assess confidence in each rea... | 176 |
| [corpusqa-10-million-token](skills/corpusqa-10-million-token/SKILL.md) | Corpus-level QA over massive document collections using memory-augmented agentic processing. Synthesize answers that req... | 179 |
| [creditaudit-2textnd-dimension-evaluation](skills/creditaudit-2textnd-dimension-evaluation/SKILL.md) | Evaluate and select LLMs using CreditAudit's 2D framework: mean ability plus stability risk (fluctuation) across system ... | 211 |
| [cross-lingual-stability-judges-under](skills/cross-lingual-stability-judges-under/SKILL.md) | Detect and fix cross-lingual evaluation instabilities in LLM-as-a-judge pipelines. Use when: 'audit my multilingual eval... | 199 |
| [darwin-dynamic-agentically-rewriting](skills/darwin-dynamic-agentically-rewriting/SKILL.md) | Evolutionary multi-agent code optimization using genetic algorithms. Agents mutate each other's training/configuration c... | 167 |
| [datacross-unified-benchmark-agent](skills/datacross-unified-benchmark-agent/SKILL.md) | Cross-modal data analysis agent that unifies structured sources (SQL, CSV, JSON) with unstructured visual documents (sca... | 203 |
| [david-vs-goliath-verifiable](skills/david-vs-goliath-verifiable/SKILL.md) | Audit and harden tool-augmented AI agent systems against Tag-Along Attacks -- adversarial agent-to-agent jailbreaks that... | 165 |
| [deepimagesearch-benchmarking-multimodal-agents](skills/deepimagesearch-benchmarking-multimodal-agents/SKILL.md) | Build agentic image retrieval systems that perform multi-step contextual reasoning over visual histories instead of isol... | 198 |
| [deepplanning-benchmarking-long-horizon-agentic](skills/deepplanning-benchmarking-long-horizon-agentic/SKILL.md) | Solve long-horizon planning tasks with verifiable constraints using the DeepPlanning methodology: proactive information ... | 155 |
| [devops-gym-benchmarking-ai-agents](skills/devops-gym-benchmarking-ai-agents/SKILL.md) | Apply the DevOps-Gym methodology to systematically tackle full-cycle DevOps tasks: build/configuration repair, runtime m... | 170 |
| [dial-summer-structured-evaluation-framework](skills/dial-summer-structured-evaluation-framework/SKILL.md) | Evaluate dialogue summaries using the DIAL-SUMMER hierarchical error taxonomy. Detects 10 fine-grained error types acros... | 244 |
| [do-truly-benefit-longer](skills/do-truly-benefit-longer/SKILL.md) | Optimize LLM context length for post-editing and refinement pipelines. Applies research showing that naively adding docu... | 252 |
| [do-vlms-have-moral](skills/do-vlms-have-moral/SKILL.md) | Audit and harden the moral robustness of Vision-Language Model (VLM) pipelines against adversarial perturbations that fl... | 211 |
| [draincode-stealthy-energy-consumption](skills/draincode-stealthy-energy-consumption/SKILL.md) | Evaluate and defend RAG-based code generation systems against energy-drain attacks that poison retrieval contexts to inf... | 221 |
| [edge-optimized-vision-language-underground-infrast](skills/edge-optimized-vision-language-underground-infrast/SKILL.md) | Build edge-deployable two-stage pipelines that combine lightweight segmentation with quantized Vision-Language Models fo... | 483 |
| [eliciting-least-to-most-reasoning-phishing](skills/eliciting-least-to-most-reasoning-phishing/SKILL.md) | Detect phishing URLs using Least-to-Most iterative decomposition with answer sensitivity scoring. Triggers: 'analyze thi... | 155 |
| [embocoach-bench-benchmarking-ai-agents](skills/embocoach-bench-benchmarking-ai-agents/SKILL.md) | | | 230 |
| [emotion-llamav2-mmeverse-framework-benchmark](skills/emotion-llamav2-mmeverse-framework-benchmark/SKILL.md) | Build multimodal emotion understanding systems using the Emotion-LLaMAv2 architecture and MMEVerse benchmark methodology... | 232 |
| [entworld-holistic-environment-benchmark](skills/entworld-holistic-environment-benchmark/SKILL.md) | Build verifiable enterprise GUI agent benchmarks using schema-grounded task generation and SQL-based deterministic verif... | 158 |
| [epistemic-context-learning-building](skills/epistemic-context-learning-building/SKILL.md) | Build trust-aware multi-agent systems using Epistemic Context Learning (ECL). Constructs peer reliability profiles from ... | 210 |
| [es-memeval-benchmarking-conversational-agents](skills/es-memeval-benchmarking-conversational-agents/SKILL.md) | Build and evaluate long-term memory systems for conversational agents using the ES-MemEval five-capability framework (in... | 226 |
| [evaluating-achieving-controllable-code](skills/evaluating-achieving-controllable-code/SKILL.md) | Instruction-guided code completion that follows user constraints on algorithm choice, data structures, control flow, and... | 197 |
| [evaluating-enhancing-vulnerability-reasoning](skills/evaluating-enhancing-vulnerability-reasoning/SKILL.md) | Perform DAG-structured vulnerability reasoning on code, modeling causal dependencies between code facts instead of linea... | 211 |
| [evaluating-kubernetes-performance-genai](skills/evaluating-kubernetes-performance-genai/SKILL.md) | Design and optimize Kubernetes-native GenAI inference platforms using Kueue job queuing, Dynamic Accelerator Slicer (DAS... | 217 |
| [evaluating-retrievalaugmented-generation-variants](skills/evaluating-retrievalaugmented-generation-variants/SKILL.md) | | | 223 |
| [evaluating-social-bias-rag](skills/evaluating-social-bias-rag/SKILL.md) | Evaluate and mitigate social bias in RAG pipelines. Use when: 'audit my RAG system for bias', 'check if retrieval introd... | 212 |
| [evaluating-they-not-know](skills/evaluating-they-not-know/SKILL.md) | Build statistically efficient LLM evaluation pipelines that combine direct accuracy with pairwise comparison signals as ... | 185 |
| [evaluation-entity-matching-recommender](skills/evaluation-entity-matching-recommender/SKILL.md) | Build and evaluate cross-dataset entity matching pipelines for recommender systems. Implements the Reddit-Amazon-EM meth... | 182 |
| [evaluation-legal-applications-challenges](skills/evaluation-legal-applications-challenges/SKILL.md) | Build evaluation pipelines for LLMs in legal tasks using a three-dimensional framework: outcome correctness, reasoning r... | 171 |
| [evaluation-oncotimia-system-supporting](skills/evaluation-oncotimia-system-supporting/SKILL.md) | Build RAG pipelines that transform unstructured clinical or domain-specific documents into structured form records using... | 213 |
| [evermembench-benchmarking-long-term-interactive](skills/evermembench-benchmarking-long-term-interactive/SKILL.md) | Build and evaluate long-term conversational memory systems for multi-party, multi-topic dialogues. Implements the EverMe... | 182 |
| [evocodebench-human-performance-benchmark-self-evol](skills/evocodebench-human-performance-benchmark-self-evol/SKILL.md) | Self-evolving code generation with iterative reflection and revision. Applies a feedback-driven loop where code is submi... | 174 |
| [featurebench-benchmarking-agentic-coding](skills/featurebench-benchmarking-agentic-coding/SKILL.md) | Extract feature-level coding tasks from repositories using test-driven dependency graph tracing. Use when the user says ... | 178 |
| [fin-rate-real-world-financial-analytics](skills/fin-rate-real-world-financial-analytics/SKILL.md) | Analyze SEC filings and financial disclosures using the Fin-RATE three-pathway methodology: detail-oriented reasoning wi... | 182 |
| [fine-tuning-gpt-5-gpu-kernel](skills/fine-tuning-gpt-5-gpu-kernel/SKILL.md) | Generate optimized GPU kernels in Triton from PyTorch reference code using the Makora RL-based iterative refinement work... | 259 |
| [flyaoc-evaluating-agentic-ontology](skills/flyaoc-evaluating-agentic-ontology/SKILL.md) | Build multi-agent systems for end-to-end ontology curation from scientific literature. Applies FlyAOC's agent architectu... | 184 |
| [focus-dllms-know-tame](skills/focus-dllms-know-tame/SKILL.md) | Deploy and optimize FOCUS inference for Diffusion Large Language Models (DLLMs). Configures attention-based token evicti... | 182 |
| [from-assistant-double-agent](skills/from-assistant-double-agent/SKILL.md) | Security audit and hardening for personalized LLM-based agents against prompt injection, tool poisoning, and memory atta... | 230 |
| [from-assumptions-actions-turning](skills/from-assumptions-actions-turning/SKILL.md) | Build uncertainty-aware planners for multi-agent systems using the PCE (Planner-Composer-Evaluator) decision tree framew... | 242 |
| [from-code-centric-concept-centric-teaching](skills/from-code-centric-concept-centric-teaching/SKILL.md) | Generate LLM-assisted coding labs that teach concepts through 'Vibe Coding' — producing working code paired with mandato... | 269 |
| [from-features-actions-explainability](skills/from-features-actions-explainability/SKILL.md) | Diagnose and explain failures in agentic AI systems using trace-based rubric evaluation, bridging static feature attribu... | 207 |
| [from-helpfulness-toxic-proactivity](skills/from-helpfulness-toxic-proactivity/SKILL.md) | Diagnose and mitigate Toxic Proactivity in LLM agent systems -- the failure mode where agents override ethical constrain... | 194 |
| [from-passive-metric-active](skills/from-passive-metric-active/SKILL.md) | Build systems that use LLM uncertainty as an active control signal -- routing computation, triggering tool calls, enabli... | 268 |
| [from-sparse-decisions-dense](skills/from-sparse-decisions-dense/SKILL.md) | Build content moderation and safety classification systems using multi-attribute trajectory reasoning instead of binary ... | 261 |
| [frost-filtering-reasoning-outliers](skills/frost-filtering-reasoning-outliers/SKILL.md) | Implement FROST (Filtering Reasoning Outliers with Attention) to prune unnecessary reasoning steps from LLM chain-of-tho... | 252 |
| [funprm-function-as-step-process-reward](skills/funprm-function-as-step-process-reward/SKILL.md) | Generate high-quality code by decomposing solutions into modular functions (Chain-of-Function style), then self-evaluati... | 251 |
| [gamedevbench-evaluating-agentic-capabilities](skills/gamedevbench-evaluating-agentic-capabilities/SKILL.md) | Agentic game development with visual feedback loops for Godot Engine projects. Applies the GameDevBench methodology: nav... | 187 |
| [generating-data-driven-reasoning-rubrics](skills/generating-data-driven-reasoning-rubrics/SKILL.md) | Build granular error taxonomies from incorrect reasoning traces, then use those rubrics to detect errors in LLM outputs ... | 170 |
| [genius-generative-fluid-intelligence](skills/genius-generative-fluid-intelligence/SKILL.md) | Evaluate and improve generative AI outputs for fluid intelligence tasks -- pattern induction from context, ad-hoc constr... | 247 |
| [gflowpo-generative-flow-network](skills/gflowpo-generative-flow-network/SKILL.md) | Optimize LLM prompts using GFlowPO's iterative generate-evaluate-refine loop with diversity-preserving exploration and d... | 171 |
| [gisa-benchmark-general-information-seeking](skills/gisa-benchmark-general-information-seeking/SKILL.md) | > | 162 |
| [gradingattack-attacking-short-answer](skills/gradingattack-attacking-short-answer/SKILL.md) | Audit LLM-based automatic short answer grading (ASAG) systems for adversarial vulnerabilities using token-level and prom... | 243 |
| [haif-human-ai-integration-framework](skills/haif-human-ai-integration-framework/SKILL.md) | Apply the HAIF protocol to organize hybrid human-AI team workflows with tiered autonomy, delegation governance, and vali... | 206 |
| [halluverse-m3-multitask-multilingual-benchmark-hal](skills/halluverse-m3-multitask-multilingual-benchmark-hal/SKILL.md) | Detect and classify hallucinations in LLM outputs across languages using the HalluVerse-M3 fine-grained taxonomy (entity... | 148 |
| [halt-hallucination-assessment-log-probs](skills/halt-hallucination-assessment-log-probs/SKILL.md) | > | 315 |
| [harnessing-precision-querying-retrieval-augmented](skills/harnessing-precision-querying-retrieval-augmented/SKILL.md) | LLM-driven precision querying of structured tabular data via Python/Pandas code generation and retrieval-augmented extra... | 163 |
| [he-snr-uncovering-latent-logic](skills/he-snr-uncovering-latent-logic/SKILL.md) | Evaluate and optimize LLM training data quality for software engineering tasks using the HE-SNR (High-Entropy Signal-to-... | 234 |
| [helm-human-centered-evaluation-framework](skills/helm-human-centered-evaluation-framework/SKILL.md) | Evaluate LLM-powered recommender systems across five human-centered dimensions: Intent Alignment, Explanation Quality, I... | 244 |
| [how-much-reasoning-retrieval-augmented](skills/how-much-reasoning-retrieval-augmented/SKILL.md) | Build contamination-aware hybrid RAG evaluation pipelines that couple knowledge graphs with text retrieval for multi-hop... | 178 |
| [humans-welcome-observe-first-look](skills/humans-welcome-observe-first-look/SKILL.md) | Analyze AI agent social network activity using topic taxonomy classification and multi-level toxicity scoring. Detects c... | 191 |
| [hunt-instead-wait-evaluating](skills/hunt-instead-wait-evaluating/SKILL.md) | > | 222 |
| [ic-eo-interpretable-code-based-assistant](skills/ic-eo-interpretable-code-based-assistant/SKILL.md) | Build conversational Earth Observation agents that turn natural-language queries into executable, auditable Python workf... | 215 |
| [icon-intent-context-coupling-multi-turn](skills/icon-intent-context-coupling-multi-turn/SKILL.md) | Build multi-turn LLM safety evaluation harnesses using the Intent-Context Coupling framework from ICON. Generates struct... | 243 |
| [ide-bench-evaluating-as-ide](skills/ide-bench-evaluating-as-ide/SKILL.md) | Apply IDE-Bench's structured agent workflow for tackling real-world software engineering tasks: systematic exploration b... | 149 |
| [improving-user-privacy-personalized](skills/improving-user-privacy-personalized/SKILL.md) | Build privacy-preserving personalized LLM systems using the P³ (Private Personalized Prediction) client-server architect... | 192 |
| [industrialized-deception-collateral-effects](skills/industrialized-deception-collateral-effects/SKILL.md) | Analyze content for AI-generated misinformation signals using the JudgeGPT/RogueGPT experimental pipeline. Evaluate text... | 216 |
| [inficoevalchain-blockchain-based-decentralized-fra](skills/inficoevalchain-blockchain-based-decentralized-fra/SKILL.md) | Design and implement decentralized LLM evaluation systems using blockchain-based consensus, multi-node validation, and s... | 202 |
| [interpreting-agentic-systems-beyond](skills/interpreting-agentic-systems-beyond/SKILL.md) | Audit and instrument agentic AI systems for system-level interpretability and accountability. Embeds traceability, causa... | 329 |
| [isd-agent-bench-comprehensive-benchmark-evaluating](skills/isd-agent-bench-comprehensive-benchmark-evaluating/SKILL.md) | Build and evaluate LLM-based Instructional Design agents using the ADDIE framework, Context Matrix scenario generation, ... | 210 |
| [jobresqa-benchmark-machine-reading](skills/jobresqa-benchmark-machine-reading/SKILL.md) | Build and evaluate multilingual machine reading comprehension systems for HR documents (resumes and job descriptions). I... | 152 |
| [just-in-time-reinforcement-learning-continual](skills/just-in-time-reinforcement-learning-continual/SKILL.md) | Implement JitRL-style continual learning for LLM agents: training-free policy optimization via experience memory, advant... | 203 |
| [kv-core-benchmarking-data-dependent-low-rank](skills/kv-core-benchmarking-data-dependent-low-rank/SKILL.md) | Benchmark and analyze KV-cache low-rank compressibility in LLMs using SVD-based evaluation and Normalized Effective Rank... | 195 |
| [layer-wise-lora-fine-tuning-similarity](skills/layer-wise-lora-fine-tuning-similarity/SKILL.md) | Selectively apply LoRA adapters to only the most important transformer layers using CKA similarity-based layer importanc... | 221 |
| [learning-decentralized-collaboration-multi-agent](skills/learning-decentralized-collaboration-multi-agent/SKILL.md) | Design and orchestrate decentralized multi-LLM collaboration systems using Multi-Agent Actor-Critic (MAAC) patterns from... | 218 |
| [learning-decode-against-compositional](skills/learning-decode-against-compositional/SKILL.md) | Detect and mitigate compositional hallucinations in video multimodal LLM outputs using triple-pathway contrastive decodi... | 284 |
| [lemur-corpus-robust-fine-tuning](skills/lemur-corpus-robust-fine-tuning/SKILL.md) | Build multilingual legal document retrieval systems by fine-tuning embedding models on domain-specific corpora with cont... | 234 |
| [leveraging-data-say-no](skills/leveraging-data-say-no/SKILL.md) | Implement memory-augmented selective prediction for vision-language models using retrieval-based confidence scoring and ... | 194 |
| [linglanmidian-systematic-evaluation-tcm](skills/linglanmidian-systematic-evaluation-tcm/SKILL.md) | Build rigorous, multi-task evaluation benchmarks for domain-specific LLMs using the LingLanMiDian methodology: synonym-t... | 245 |
| [lingua-safetybench-benchmark-safety-evaluation-mul](skills/lingua-safetybench-benchmark-safety-evaluation-mul/SKILL.md) | Evaluate and stress-test multilingual vision-language model safety using the Lingua-SafetyBench methodology. Constructs ... | 160 |
| [lingxidiagbench-multi-agent-framework-benchmarking](skills/lingxidiagbench-multi-agent-framework-benchmarking/SKILL.md) | Build multi-agent benchmarking systems with role-separated agents (simulator, interviewer, evaluator) for structured mul... | 216 |
| [livemedbench-contamination-free-medical-benchmark](skills/livemedbench-contamination-free-medical-benchmark/SKILL.md) | Build contamination-free LLM evaluation pipelines with multi-agent data curation and automated rubric-based scoring. Use... | 296 |
| [livibench-omnimodal-benchmark-interactive](skills/livibench-omnimodal-benchmark-interactive/SKILL.md) | Build omnimodal benchmarks and evaluation pipelines for interactive video understanding (livestreams, real-time comments... | 238 |
| [llm-prompt-evaluation-educational](skills/llm-prompt-evaluation-educational/SKILL.md) | Systematically design, evaluate, and rank LLM prompts for educational applications using tournament-style Glicko-2 compa... | 223 |
| [loca-bench-benchmarking-agents-under](skills/loca-bench-benchmarking-agents-under/SKILL.md) | Apply context management strategies from LOCA-bench to prevent context rot in long-running agent tasks. Implements progr... | 169 |
| [locomo-plus-beyond-factual-cognitive-memory](skills/locomo-plus-beyond-factual-cognitive-memory/SKILL.md) | Build and evaluate cognitive memory systems for LLM dialogue agents that retain implicit user constraints (state, goals,... | 216 |
| [logicscore-fine-grained-logic-evaluation](skills/logicscore-fine-grained-logic-evaluation/SKILL.md) | Evaluate the logical integrity of LLM-generated multi-hop answers using Horn Rule backward chaining. Scores Completeness... | 185 |
| [lost-translation-comparative-study](skills/lost-translation-comparative-study/SKILL.md) | Cross-lingual safety evaluation for LLMs using the CompositeHarm methodology. Builds multilingual safety benchmarks that... | 176 |
| [lps-bench-benchmarking-safety-awareness](skills/lps-bench-benchmarking-safety-awareness/SKILL.md) | > | 191 |
| [mad-modality-adaptive-decoding-mitigating](skills/mad-modality-adaptive-decoding-mitigating/SKILL.md) | Implement Modality-Adaptive Decoding (MAD) to suppress cross-modal hallucinations in multimodal LLMs. Uses self-assessme... | 227 |
| [made-benchmark-environments-closed-loop](skills/made-benchmark-environments-closed-loop/SKILL.md) | Build closed-loop discovery benchmarks where an agent iteratively proposes, evaluates, and refines candidates under a fi... | 144 |
| [malloc-benchmarking-memory-aware-long](skills/malloc-benchmarking-memory-aware-long/SKILL.md) | Apply memory-aware compression strategies to long-sequence recommendation systems. Benchmark KV-cache compression techni... | 248 |
| [mas-prove-understanding-process-verification](skills/mas-prove-understanding-process-verification/SKILL.md) | Design and implement process verification for multi-agent LLM systems. Add intermediate-step evaluation to multi-agent w... | 237 |
| [masalbench-benchmark-contextual-cross-cultural](skills/masalbench-benchmark-contextual-cross-cultural/SKILL.md) | Build cross-cultural figurative language benchmarks and evaluation pipelines for LLMs. Applies the MasalBench methodolog... | 186 |
| [massive-sound-embedding-benchmark](skills/massive-sound-embedding-benchmark/SKILL.md) | | | 236 |
| [mcp-atlas-large-scale-benchmark-tool-use](skills/mcp-atlas-large-scale-benchmark-tool-use/SKILL.md) | Design and evaluate multi-server MCP tool-use benchmarks using claims-based scoring rubrics. Use when: 'benchmark my MCP... | 210 |
| [meetbench-xl-calibrated-multidimensional-evaluation](skills/meetbench-xl-calibrated-multidimensional-evaluation/SKILL.md) | | | 191 |
| [memadapter-fast-alignment-across](skills/memadapter-fast-alignment-across/SKILL.md) | Unify heterogeneous agent memory systems (explicit graphs, parametric weights, latent KV-caches) via generative subgraph... | 166 |
| [merlin-discovery-engine-photonic](skills/merlin-discovery-engine-photonic/SKILL.md) | Build and benchmark photonic quantum machine learning models using the MerLin framework. Integrates linear optical circu... | 248 |
| [mermaid-memory-enhanced-retrieval-reasoning](skills/mermaid-memory-enhanced-retrieval-reasoning/SKILL.md) | Memory-enhanced multi-agent retrieval and reasoning for veracity assessment and fact-checking. Use when: 'verify this cl... | 189 |
| [mhdash-online-platform-benchmarking](skills/mhdash-online-platform-benchmarking/SKILL.md) | Build risk-aware evaluation pipelines for mental health AI assistants using the MHDash framework. Implements multi-dimen... | 285 |
| [mmr-bench-comprehensive-benchmark-multimodal](skills/mmr-bench-comprehensive-benchmark-multimodal/SKILL.md) | Build cost-aware multimodal LLM routing systems that select the best model per query based on input signals, budget cons... | 175 |
| [mmts-bench-comprehensive-benchmark-time](skills/mmts-bench-comprehensive-benchmark-time/SKILL.md) | Evaluate and improve LLM performance on time series question-answering using the MMTS-BENCH hierarchical taxonomy. Cover... | 193 |
| [modality-gap-driven-subspace-alignment](skills/modality-gap-driven-subspace-alignment/SKILL.md) | Align multimodal embeddings (vision-language) by correcting the modality gap using the ReAlign/ReVision technique. Fixes... | 231 |
| [mpib-benchmark-medical-prompt](skills/mpib-benchmark-medical-prompt/SKILL.md) | Evaluate and defend clinical LLM systems against prompt injection attacks using the MPIB benchmark methodology. Implemen... | 177 |
| [mrag-benchmarking-retrieval-augmented-generation](skills/mrag-benchmarking-retrieval-augmented-generation/SKILL.md) | Build and evaluate biomedical RAG pipelines using the MRAG benchmark methodology. Configures retrieval, prompting, and g... | 183 |
| [multi-targeted-graph-backdoor-attack](skills/multi-targeted-graph-backdoor-attack/SKILL.md) | Implement and analyze multi-targeted backdoor attacks on Graph Neural Networks (GNNs) using subgraph injection. Use when... | 193 |
| [naamse-framework-evolutionary-security](skills/naamse-framework-evolutionary-security/SKILL.md) | Implement evolutionary security evaluation for AI agents using the NAAMSE framework — genetic prompt mutation, hierarchi... | 201 |
| [now-you-hear-me](skills/now-you-hear-me/SKILL.md) | Audit and defend large audio-language models (LALMs) against narrative-style audio jailbreaks. Based on the 'Now You Hea... | 311 |
| [odysseyarena-benchmarking-long-horizon-active](skills/odysseyarena-benchmarking-long-horizon-active/SKILL.md) | Design and run inductive agent benchmarks where LLMs must discover hidden rules through long-horizon interaction loops r... | 176 |
| [omni-rrm-advancing-omni-reward](skills/omni-rrm-advancing-omni-reward/SKILL.md) | Build rubric-grounded reward models and preference evaluation pipelines for multimodal AI outputs. Use when asked to 'ev... | 180 |
| [omnicode-benchmark-evaluating-software](skills/omnicode-benchmark-evaluating-software/SKILL.md) | Evaluate and improve code across four software engineering dimensions: bug fixing, test generation, code review fixing, ... | 199 |
| [omnireview-large-scale-benchmark-llm-enhanced](skills/omnireview-large-scale-benchmark-llm-enhanced/SKILL.md) | Build reviewer/expert recommendation systems using LLM-generated semantic profiles and Multi-gate Mixture-of-Experts (MM... | 204 |
| [on-effectiveness-llm-specific-fine-tuning](skills/on-effectiveness-llm-specific-fine-tuning/SKILL.md) | Build and evaluate AI-generated text detectors using LLM-specific fine-tuning strategies. Covers corpus construction, pe... | 171 |
| [on-use-generate-dataset](skills/on-use-generate-dataset/SKILL.md) | Generate diverse, validated datasets of neural network implementations using LLM-driven combinatorial design. Use when: ... | 195 |
| [openguandan-large-scale-imperfect-information](skills/openguandan-large-scale-imperfect-information/SKILL.md) | Build AI agents for the OpenGuanDan imperfect-information card game benchmark. Covers WebSocket client implementation, g... | 366 |
| [opinf-llm-parametric-pde-solving](skills/opinf-llm-parametric-pde-solving/SKILL.md) | > | 200 |
| [opportunities-aiml-rubin-lsst](skills/opportunities-aiml-rubin-lsst/SKILL.md) | Build trustworthy ML pipelines for large-scale scientific data analysis with calibrated uncertainties, simulation-based ... | 249 |
| [optimal-turkish-subword-strategies](skills/optimal-turkish-subword-strategies/SKILL.md) | Design and evaluate subword tokenizers for Turkish and other morphologically rich languages (MRLs) using the vocabulary-... | 253 |
| [outrunning-cutoffs-live-kernel](skills/outrunning-cutoffs-live-kernel/SKILL.md) | Build live-evolving kernel crash resolution benchmarks and agent environments using the Live-kBench/kEnv methodology. Se... | 153 |
| [parse-open-domain-reasoning-question](skills/parse-open-domain-reasoning-question/SKILL.md) | Build and evaluate reasoning-focused QA systems for low-resource languages using the PARSE methodology: structured promp... | 220 |
| [pathwise-planning-world-automated](skills/pathwise-planning-world-automated/SKILL.md) | Multi-agent heuristic design framework that uses an entailment graph, policy/world-model/critic agents, and routed refle... | 165 |
| [peerrank-autonomous-evaluation-web-grounded](skills/peerrank-autonomous-evaluation-web-grounded/SKILL.md) | Implement PeerRank-style autonomous multi-model evaluation pipelines where LLMs symmetrically generate tasks, answer wit... | 215 |
| [persodpo-scalable-preference-optimization](skills/persodpo-scalable-preference-optimization/SKILL.md) | Build scalable preference optimization pipelines for persona-grounded dialogue systems using multi-LLM evaluation. Use w... | 173 |
| [persona-jailbreaking](skills/persona-jailbreaking/SKILL.md) | Audit and defend LLM-powered applications against persona manipulation attacks using the PHISH framework (Persona Hijack... | 301 |
| [phostream-benchmarking-real-world-streaming](skills/phostream-benchmarking-real-world-streaming/SKILL.md) | Build streaming multimodal benchmarks and evaluate omnimodal assistants on continuous audio-visual input with temporal r... | 238 |
| [precise-reducing-bias-evaluations](skills/precise-reducing-bias-evaluations/SKILL.md) | Implement the PRECISE framework to debias LLM-as-judge evaluations of search, ranking, and RAG systems by combining a sm... | 212 |
| [privacy-collapse-benign-fine-tuning](skills/privacy-collapse-benign-fine-tuning/SKILL.md) | Audit fine-tuning datasets and pipelines for privacy collapse — the silent failure where benign training data degrades a... | 195 |
| [proopf-benchmarking-improving-professional-grade](skills/proopf-benchmarking-improving-professional-grade/SKILL.md) | Translate natural-language power system operational requirements into executable Optimal Power Flow (OPF) optimization c... | 218 |
| [protean-compiler-agile-framework](skills/protean-compiler-agile-framework/SKILL.md) | Guide fine-grained LLVM compiler phase ordering using the Protean framework's agile optimization approach — clustering p... | 149 |
| [proxywar-dynamic-assessment-of](skills/proxywar-dynamic-assessment-of/SKILL.md) | Build competitive game-arena evaluation frameworks for LLM-generated code using ProxyWar's multi-layer pipeline: agent g... | 197 |
| [quasar-universal-autonomous-system](skills/quasar-universal-autonomous-system/SKILL.md) | Build autonomous multi-scale scientific simulation pipelines using the QUASAR architecture: a Strategist-Operator-Evalua... | 165 |
| [raca-representation-aware-coverage-criteria](skills/raca-representation-aware-coverage-criteria/SKILL.md) | Evaluate and improve LLM safety test suites using representation-aware coverage criteria. Implements the RACA framework ... | 242 |
| [ral-bench-benchmarking-application-level-functiona](skills/ral-bench-benchmarking-application-level-functiona/SKILL.md) | Generate and evaluate complete multi-file application repositories with both functional correctness and non-functional q... | 179 |
| [realsec-bench-benchmark-evaluating-secure](skills/realsec-bench-benchmark-evaluating-secure/SKILL.md) | > | 195 |
| [rebel-hidden-knowledge-recovery](skills/rebel-hidden-knowledge-recovery/SKILL.md) | Machine unlearning for LLMs aims to remove sensitive or copyrighted data from trained models. Implements techniques from... | 28 |
| [rethinking-llm-as-a-judge-representation-as-a-judg](skills/rethinking-llm-as-a-judge-representation-as-a-judg/SKILL.md) | Build probing-based evaluation pipelines that judge LLM output quality using hidden-state representations from small lan... | 160 |
| [rethinking-scientific-modeling-physically](skills/rethinking-scientific-modeling-physically/SKILL.md) | Generate physics-consistent, simulation-executable structural engineering code using constraint-oriented alignment and v... | 209 |
| [robustexplain-evaluating-robustness-llm-based](skills/robustexplain-evaluating-robustness-llm-based/SKILL.md) | Evaluate robustness of LLM-generated recommendation explanations under realistic user behavior noise. Use when: 'test ex... | 211 |
| [rubberduckbench-benchmark-ai-coding](skills/rubberduckbench-benchmark-ai-coding/SKILL.md) | Evaluate and improve AI coding assistant responses using RubberDuckBench's rubric-based methodology. Detects hallucinati... | 182 |
| [rvcbench-benchmarking-robustness-voice](skills/rvcbench-benchmarking-robustness-voice/SKILL.md) | Benchmark and harden voice cloning systems against real-world degradation using the RVCBench framework. Evaluates VC mod... | 164 |
| [scratcheval-multimodal-evaluation-framework](skills/scratcheval-multimodal-evaluation-framework/SKILL.md) | Evaluate, debug, and repair block-based Scratch programs using a three-layer executable protocol (VM execution, block-le... | 156 |
| [se-bench-benchmarking-self-evolution-knowledge](skills/se-bench-benchmarking-self-evolution-knowledge/SKILL.md) | Design knowledge-internalization benchmarks and closed-book training pipelines for LLM self-evolution. Use when: 'build ... | 163 |
| [seccodeprm-process-reward-code](skills/seccodeprm-process-reward-code/SKILL.md) | Step-level security scoring for code generation and vulnerability detection using process reward model techniques. Use w... | 201 |
| [self-evolving-recommendation-system-end-to-end](skills/self-evolving-recommendation-system-end-to-end/SKILL.md) | Build autonomous ML optimization pipelines that use LLM agents to generate, evaluate, and deploy model improvements in a... | 168 |
| [self-improving-pretraining-post-trained-pretrain](skills/self-improving-pretraining-post-trained-pretrain/SKILL.md) | Build data curation pipelines that use a strong post-trained model to score, filter, and rewrite pretraining corpora for... | 256 |
| [shopsimulator-evaluating-exploring-rl-driven](skills/shopsimulator-evaluating-exploring-rl-driven/SKILL.md) | Build and evaluate LLM-based shopping assistant agents using structured multi-turn dialogue, personalized product search... | 285 |
| [socialveil-probing-social-intelligence](skills/socialveil-probing-social-intelligence/SKILL.md) | Stress-test LLM agents' social intelligence by injecting realistic communication barriers (semantic vagueness, sociocult... | 203 |
| [socratic-geo-synthetic-data-generation](skills/socratic-geo-synthetic-data-generation/SKILL.md) | Generate high-quality synthetic training data through multi-agent feedback loops where a Teacher agent creates parameter... | 226 |
| [sonic-o1-real-world-benchmark-evaluating](skills/sonic-o1-real-world-benchmark-evaluating/SKILL.md) | Evaluate multimodal LLMs on audio-video understanding using the SONIC-O1 benchmark framework. Covers three task types: v... | 238 |
| [sparse-sparse-safety-unsafe](skills/sparse-sparse-safety-unsafe/SKILL.md) | Audit and harden Mixture-of-Experts (MoE) LLM deployments against unsafe routing vulnerabilities using RoSais scoring an... | 184 |
| [sparseeval-evaluation-sparse-optimization](skills/sparseeval-evaluation-sparse-optimization/SKILL.md) | Efficiently evaluate LLMs on benchmarks by selecting a small subset of anchor items via sparse optimization, reproducing... | 221 |
| [standardizing-longitudinal-radiology-report](skills/standardizing-longitudinal-radiology-report/SKILL.md) | Build LLM-based pipelines that automatically detect and classify longitudinal (temporal) changes in radiology reports. U... | 230 |
| [steereval-framework-evaluating-steerability](skills/steereval-framework-evaluating-steerability/SKILL.md) | Evaluate and improve the steerability of natural-language-profile-based recommender systems using the SteerEval framewor... | 191 |
| [steuerllm-local-specialized-german](skills/steuerllm-local-specialized-german/SKILL.md) | Build domain-specialized LLM pipelines for formal-rule domains (tax law, legal, regulatory) using retrieval-augmented sy... | 203 |
| [strong-reasoning-isnt-enough](skills/strong-reasoning-isnt-enough/SKILL.md) | Build interactive diagnostic agents that systematically elicit evidence before concluding, using the REFINE (Reasoning-E... | 251 |
| [supchain-bench-benchmarking-real-world-supply](skills/supchain-bench-benchmarking-real-world-supply/SKILL.md) | Build reliable long-horizon supply chain agents using the SupChain-ReAct pattern: multi-path ReAct trajectories with maj... | 198 |
| [swe-agi-benchmarking-specification-driven-software](skills/swe-agi-benchmarking-specification-driven-software/SKILL.md) | Build production-scale software systems from formal specifications, RFCs, and standards documents using specification-dr... | 189 |
| [swe-context-bench-benchmark](skills/swe-context-bench-benchmark/SKILL.md) | Reuse prior coding experience across related repository tasks. Accumulate, summarize, retrieve, and inject compact exper... | 180 |
| [swe-refactor-repository-level-benchmark-real-world](skills/swe-refactor-repository-level-benchmark-real-world/SKILL.md) | > | 151 |
| [sycoeval-em-sycophancy-evaluation-simulated](skills/sycoeval-em-sycophancy-evaluation-simulated/SKILL.md) | Build multi-agent adversarial simulations to evaluate LLM sycophancy and policy compliance under social pressure. Use wh... | 241 |
| [tam-eval-evaluating-llms-for](skills/tam-eval-evaluating-llms-for/SKILL.md) | | | 225 |
| [tamperbench-systematically-stress-testing-safety](skills/tamperbench-systematically-stress-testing-safety/SKILL.md) | Set up and run TamperBench pipelines to systematically stress-test LLM safety under fine-tuning and tampering attacks. U... | 250 |
| [tangrampuzzle-evaluating-multimodal-compositional](skills/tangrampuzzle-evaluating-multimodal-compositional/SKILL.md) | Evaluate and build compositional spatial reasoning systems using geometry-grounded benchmarks and symbolic coordinate fr... | 233 |
| [teaching-evaluating-reason-about](skills/teaching-evaluating-reason-about/SKILL.md) | Apply knowledge-augmented reasoning distillation for polymer design tasks. Builds structured Chain-of-Thought pipelines ... | 202 |
| [testexplora-benchmarking-proactive-bug](skills/testexplora-benchmarking-proactive-bug/SKILL.md) | > | 218 |
| [the-clef-2026-finmmeval-lab](skills/the-clef-2026-finmmeval-lab/SKILL.md) | Build multilingual, multimodal financial AI evaluation pipelines using the FinMMEval framework. Covers financial exam QA... | 246 |
| [the-compliance-paradox-semantic-instruction](skills/the-compliance-paradox-semantic-instruction/SKILL.md) | Detect and defend against adversarial prompt injections hidden in code submissions that exploit LLM instruction-followin... | 229 |
| [the-landscape-prompt-injection](skills/the-landscape-prompt-injection/SKILL.md) | Harden LLM agent systems against prompt injection using layered text/model/execution defenses and the AgentPI evaluation... | 244 |
| [the-necessity-unified-framework](skills/the-necessity-unified-framework/SKILL.md) | Design and implement standardized, reproducible evaluation harnesses for LLM-based agents. Eliminates confounding factor... | 185 |
| [the-shadow-self-intrinsic](skills/the-shadow-self-intrinsic/SKILL.md) | Detect and mitigate intrinsic value misalignment in LLM agent systems using the IMPRESS scenario-driven framework. Use w... | 234 |
| [timeblind-spatio-temporal-compositionality-benchma](skills/timeblind-spatio-temporal-compositionality-benchma/SKILL.md) | Build and evaluate spatio-temporal reasoning benchmarks for video LLMs using the TimeBlind minimal-pairs methodology. Ge... | 233 |
| [timemachine-bench-benchmark-evaluating-capabilitie](skills/timemachine-bench-benchmark-evaluating-capabilitie/SKILL.md) | Systematic dependency migration for Python projects. Diagnose and fix test failures caused by dependency updates using a... | 143 |
| [towards-adaptive-scalable-robust](skills/towards-adaptive-scalable-robust/SKILL.md) | Implement RAPS (Reputation-Aware Publish-Subscribe) multi-agent coordination using intent-based pub/sub messaging, react... | 205 |
| [towards-ai-evaluation-domain-specific](skills/towards-ai-evaluation-domain-specific/SKILL.md) | Build and evaluate domain-specific RAG systems with iterative user-feedback refinement, source grounding, and structured... | 260 |
| [trailblazer-history-guided-reinforcement-learning](skills/trailblazer-history-guided-reinforcement-learning/SKILL.md) | Build history-aware RL pipelines for multi-turn LLM red-teaming and safety evaluation. Implements attention-weighted int... | 244 |
| [trapped-past-disentangling-fluid](skills/trapped-past-disentangling-fluid/SKILL.md) | Diagnose whether an LLM is memorizing or reasoning by constructing distributional proximity tests. Classifies task input... | 174 |
| [tricky2-benchmark-evaluating-human-error](skills/tricky2-benchmark-evaluating-human-error/SKILL.md) | Taxonomy-guided analysis of mixed human+LLM bugs in code. Classifies bug origins, localizes interacting defects, and rep... | 207 |
| [triplay-rl-tri-role-self-play-reinforcement](skills/triplay-rl-tri-role-self-play-reinforcement/SKILL.md) | Apply the TriPlay-RL tri-role adversarial self-play framework to systematically red-team, harden, and evaluate LLM-power... | 208 |
| [tsrbench-comprehensive-multi-task-multi-modal](skills/tsrbench-comprehensive-multi-task-multi-modal/SKILL.md) | Evaluate and build multi-modal time series reasoning pipelines using the TSRBench framework. Covers perception, reasonin... | 206 |
| [tutorial-reasoning-ir-ir](skills/tutorial-reasoning-ir-ir/SKILL.md) | Build reasoning-enhanced information retrieval pipelines that go beyond semantic matching. Applies five methodological f... | 249 |
| [uncertainty-and-fairness-awareness](skills/uncertainty-and-fairness-awareness/SKILL.md) | Audit LLM-based recommendation systems for predictive uncertainty and demographic fairness bias. Implements the SNSR/SNS... | 230 |
| [unicomp-unified-evaluation-compression](skills/unicomp-unified-evaluation-compression/SKILL.md) | Guide Claude through evaluating and recommending LLM compression strategies (pruning, quantization, distillation) using ... | 177 |
| [unikie-bench-benchmarking-multimodal-key](skills/unikie-bench-benchmarking-multimodal-key/SKILL.md) | Extract structured key information from document images using schema-guided prompting for LMMs. Builds KIE pipelines tha... | 292 |
| [unveiling-cognitive-compass-theory-of-mind-guided](skills/unveiling-cognitive-compass-theory-of-mind-guided/SKILL.md) | Apply Theory-of-Mind (ToM) guided reasoning chains to multimodal emotion analysis tasks. Decomposes emotional reasoning ... | 197 |
| [urdubench-urdu-reasoning-benchmark](skills/urdubench-urdu-reasoning-benchmark/SKILL.md) | Build high-quality reasoning benchmarks for Urdu and other low-resource languages using contextually ensembled translati... | 171 |
| [use-graph-it-needs](skills/use-graph-it-needs/SKILL.md) | Implement adaptive RAG pipelines that route queries to dense retrieval, graph-based retrieval, or a weighted fusion base... | 254 |
| [valueflow-measuring-propagation-value](skills/valueflow-measuring-propagation-value/SKILL.md) | Measure and analyze how value perturbations propagate through multi-agent LLM systems using the ValueFlow framework. Dia... | 180 |
| [vectra-metric-dataset-visual](skills/vectra-metric-dataset-visual/SKILL.md) | Assess visual quality of translated product images using Vectra's 14-dimension scoring framework. Use when: 'evaluate tr... | 305 |
| [vespo-variational-sequence-level-soft](skills/vespo-variational-sequence-level-soft/SKILL.md) | Implement VESPO (Variational Sequence-Level Soft Policy Optimization) for stable off-policy LLM reinforcement learning. ... | 189 |
| [videostf-stress-testing-output-repetition](skills/videostf-stress-testing-output-repetition/SKILL.md) | Detect and stress-test output repetition in Video Large Language Models using n-gram metrics and temporal perturbations ... | 295 |
| [vision-deepresearch-benchmark-rethinking-visual-te](skills/vision-deepresearch-benchmark-rethinking-visual-te/SKILL.md) | Build and evaluate Vision-DeepResearch pipelines that combine cropped visual search with multi-hop textual search for ro... | 216 |
| [visual-cognitive-demands-model-powered](skills/visual-cognitive-demands-model-powered/SKILL.md) | Evaluate visual and cognitive demands of in-vehicle LLM interfaces using the Monk et al. (2026) dual-metric framework. I... | 290 |
| [vlm-guided-iterative-refinement-surgical](skills/vlm-guided-iterative-refinement-surgical/SKILL.md) | Build iterative VLM-guided refinement pipelines for image segmentation tasks, especially surgical/medical imagery. Uses ... | 255 |
| [voxmorph-scalable-zero-shot-voice](skills/voxmorph-scalable-zero-shot-voice/SKILL.md) | Build and deploy zero-shot voice identity morphing pipelines using disentangled prosody/timbre embeddings and Spherical ... | 203 |
| [whats-benchmark-case-swe-bench-automated](skills/whats-benchmark-case-swe-bench-automated/SKILL.md) | > | 318 |
| [when-better-prompts-hurt](skills/when-better-prompts-hurt/SKILL.md) | Evaluation-driven prompt iteration using the Define-Test-Diagnose-Fix loop and Minimum Viable Evaluation Suite (MVES). P... | 196 |
| [when-evaluation-becomes-side](skills/when-evaluation-becomes-side/SKILL.md) | Detect and mitigate regime leakage in AI systems -- the information-theoretic vulnerability where models distinguish eva... | 237 |
| [whitespaces-dont-lie-feature-driven](skills/whitespaces-dont-lie-feature-driven/SKILL.md) | Detect whether source code was written by a human or generated by an AI (ChatGPT, Copilot, etc.) using whitespace, inden... | 255 |
| [who-gets-which-message](skills/who-gets-which-message/SKILL.md) | Audit demographic bias in LLM-generated targeted text. Detects age- and gender-based stereotyping in personalized messag... | 221 |
| [wiki-live-challenge-challenging](skills/wiki-live-challenge-challenging/SKILL.md) | Evaluate deep research agents and LLM-generated long-form articles using the Wiki Live Challenge framework: 39 fine-grai... | 264 |
| [will-it-survive-deciphering](skills/will-it-survive-deciphering/SKILL.md) | Analyze the survival and maintenance fate of AI-generated code in repositories using survival analysis techniques from R... | 173 |
| [world-workflows-benchmark-bringing](skills/world-workflows-benchmark-bringing/SKILL.md) | Build world models for enterprise systems with hidden workflows and cascading database effects. Applies the probe-observ... | 185 |
| [zero-shot-product-attribute-labeling](skills/zero-shot-product-attribute-labeling/SKILL.md) | Extract and classify product attributes from images using Vision-Language Models with structured prompts and a three-tie... | 268 |
| [zero2text-zero-training-cross-domain-inversion](skills/zero2text-zero-training-cross-domain-inversion/SKILL.md) | Implement embedding inversion attacks that reconstruct original text from vector embeddings without training data, using... | 188 |

---

## Efficiency & Optimization

**260 skills**

| Skill | Description | Lines |
|-------|-------------|-------|
| [1100-high-efficiency-visual-adapter-complex](skills/1100-high-efficiency-visual-adapter-complex/SKILL.md) | Implement CoLin (Complex Linear Projection) adapters for parameter-efficient fine-tuning of vision foundation models. Ad... | 251 |
| [a2rag-adaptive-agentic-graph](skills/a2rag-adaptive-agentic-graph/SKILL.md) | Build adaptive, cost-aware Graph-RAG pipelines that route queries through escalating retrieval stages (local -> bridge -... | 229 |
| [accelerating-social-science-research](skills/accelerating-social-science-research/SKILL.md) | Implement the EXPERIGEN agentic framework for automated hypothesis generation and empirical validation on datasets. Uses... | 177 |
| [acegrpo-adaptive-curriculum-group](skills/acegrpo-adaptive-curriculum-group/SKILL.md) | Adaptive curriculum-driven iterative optimization for autonomous ML engineering tasks. Uses Evolving Data Buffers and Le... | 220 |
| [affective-flow-emotional-support](skills/affective-flow-emotional-support/SKILL.md) | Build emotionally supportive multi-turn conversation systems using the AFlow framework — affective flow modeling with MC... | 216 |
| [agentark-distilling-multi-agent-intelligence](skills/agentark-distilling-multi-agent-intelligence/SKILL.md) | Distill multi-agent debate reasoning into a single LLM's behavior. Apply AgentArk's three-tier distillation strategy to ... | 199 |
| [aiano-enhancing-information-retrieval](skills/aiano-enhancing-information-retrieval/SKILL.md) | Build AI-augmented annotation pipelines for creating high-quality information retrieval and QA datasets. Combines LLM-ge... | 161 |
| [an-cost-efficient-agentic-framework](skills/an-cost-efficient-agentic-framework/SKILL.md) | Audit Ethereum smart contracts for business logic vulnerabilities using Heimdallr's four-phase agentic pipeline: functio... | 202 |
| [attn-gs-attention-guided-context-compression](skills/attn-gs-attention-guided-context-compression/SKILL.md) | Compress long user contexts (profiles, histories, documents) into concise, high-quality summaries using attention-guided... | 158 |
| [audiorouter-data-audio-understanding](skills/audiorouter-data-audio-understanding/SKILL.md) | Build audio understanding systems that route between internal LLM reasoning and external audio tools using a lightweight... | 249 |
| [autonomous-chain-of-thought-distillation-graph-bas](skills/autonomous-chain-of-thought-distillation-graph-bas/SKILL.md) | Implement FraudCoT-style graph-aware chain-of-thought distillation for fraud detection on text-attributed graphs. Combin... | 199 |
| [avere-improving-audiovisual-emotion](skills/avere-improving-audiovisual-emotion/SKILL.md) | Build emotion-aware multimodal AI systems that resist spurious cue-emotion associations and hallucinated audiovisual evi... | 177 |
| [bayesflow-probability-inference-framework](skills/bayesflow-probability-inference-framework/SKILL.md) | Generate high-quality multi-step LLM workflows using Bayesian inference with parallel look-ahead rollouts and importance... | 204 |
| [bear-beam-search-aware-optimization-recommendation](skills/bear-beam-search-aware-optimization-recommendation/SKILL.md) | Implement BEAR (Beam-SEarch-Aware Regularization) to fix training-inference mismatch in LLM-based recommendation systems... | 220 |
| [better-as-generators-than](skills/better-as-generators-than/SKILL.md) | Generate synthetic labeled datasets with LLMs to train smaller, cheaper classifiers -- especially for low-resource langu... | 178 |
| [beyond-accuracy-cognitive-load](skills/beyond-accuracy-cognitive-load/SKILL.md) | Analyze and reduce cognitive load in tool-use agent workflows using the Cognitive Load Framework from AAAI 2026. Diagnos... | 218 |
| [beyond-alignment-expanding-reasoning](skills/beyond-alignment-expanding-reasoning/SKILL.md) | Apply Manifold-Reshaping Policy Optimization (MRPO) to expand LLM reasoning capacity beyond alignment. Implements spectr... | 250 |
| [binaryppo-policy-optimization-binary](skills/binaryppo-policy-optimization-binary/SKILL.md) | Implement BinaryPPO, an offline RL framework that reformulates binary classification as reward maximization with confide... | 173 |
| [bridging-academia-industry-comprehensive](skills/bridging-academia-industry-comprehensive/SKILL.md) | Attributed graph clustering pipelines using PyAGC's Encode-Cluster-Optimize framework. Triggers: 'cluster nodes in a gra... | 264 |
| [bridging-modality-gap-roadside](skills/bridging-modality-gap-roadside/SKILL.md) | Build training-free pipelines that convert sparse 3D LiDAR point clouds into depth-encoded 2D images for classification ... | 211 |
| [c-mop-integrating-momentum-boundary-aware](skills/c-mop-integrating-momentum-boundary-aware/SKILL.md) | Optimize LLM system prompts iteratively using boundary-aware contrastive sampling and momentum-guided clustering from th... | 179 |
| [cam-causality-based-analysis-framework](skills/cam-causality-based-analysis-framework/SKILL.md) | Analyze and optimize multi-agent code generation pipelines using causality-based importance ranking of intermediate feat... | 153 |
| [can-small-handle-context-summarized](skills/can-small-handle-context-summarized/SKILL.md) | Build context-summarized multi-turn QA systems that let small language models (SLMs) handle customer-service dialogues w... | 254 |
| [can-vision-language-handle-long-context](skills/can-vision-language-handle-long-context/SKILL.md) | Apply visual code compression (LongCodeOCR) to handle long-context code analysis with Vision-Language Models. Renders so... | 185 |
| [chehab-rl-learning-optimize](skills/chehab-rl-learning-optimize/SKILL.md) | Optimize Fully Homomorphic Encryption code using RL-guided rewriting rules for automatic vectorization, latency reductio... | 206 |
| [chunking-retrieval-re-ranking-empirical-evaluation](skills/chunking-retrieval-re-ranking-empirical-evaluation/SKILL.md) | Build and optimize two-stage RAG pipelines with bi-encoder retrieval, cross-encoder re-ranking, and empirically-validate... | 230 |
| [cimrag-cim-aware-domain-adaptive-noise-resilient](skills/cimrag-cim-aware-domain-adaptive-noise-resilient/SKILL.md) | Build noise-resilient RAG retrieval pipelines for edge/resource-constrained deployments. Implements TONEL (Task-Oriented... | 227 |
| [clustering-driven-memory-compression-on-device](skills/clustering-driven-memory-compression-on-device/SKILL.md) | | | 254 |
| [codeocr-effectiveness-vision-code](skills/codeocr-effectiveness-vision-code/SKILL.md) | Render source code as images for vision LLM processing to reduce token cost while preserving understanding. Use when: 'r... | 241 |
| [cofrgenet-continued-fraction-architectures](skills/cofrgenet-continued-fraction-architectures/SKILL.md) | Implement Continued Fraction Generative Networks (CoFrGeNets) -- parameter-efficient replacements for Multi-head Attenti... | 267 |
| [colt-lightweight-multi-llm-collaboration](skills/colt-lightweight-multi-llm-collaboration/SKILL.md) | | | 190 |
| [comet-collaborative-memory-transformer](skills/comet-collaborative-memory-transformer/SKILL.md) | Design and implement dual-memory chunk-based architectures for efficient long-context LLM processing. Use when asked abo... | 199 |
| [comprehensive-comparison-rag-methods](skills/comprehensive-comparison-rag-methods/SKILL.md) | Select and configure the right RAG strategy for conversational QA systems based on dataset characteristics. Use when: 'b... | 167 |
| [contextevolve-multi-agent-context-compression](skills/contextevolve-multi-agent-context-compression/SKILL.md) | Multi-agent iterative code optimization using context compression. Decomposes optimization into three agents (Summarizer... | 175 |
| [controlling-output-rankings-generative](skills/controlling-output-rankings-generative/SKILL.md) | Optimize product/content descriptions to influence rankings in LLM-based search engines (generative engines) using the C... | 245 |
| [cord-bridging-audio-text-reasoning](skills/cord-bridging-audio-text-reasoning/SKILL.md) | Implement CORD (Cross-modal On-policy Distillation) to bridge modality gaps in multimodal AI systems. Applies weighted s... | 238 |
| [cost-efficient-rag-entity-matching](skills/cost-efficient-rag-entity-matching/SKILL.md) | Build cost-efficient RAG pipelines for entity matching and deduplication using blocking-based batch retrieval and genera... | 187 |
| [cowork-x-experience-optimized-co-evolution-multi-a](skills/cowork-x-experience-optimized-co-evolution-multi-a/SKILL.md) | Build multi-agent collaboration systems with experience-driven co-evolution using HTN skill libraries and post-episode o... | 150 |
| [ctrlcot-dual-granularity-chain-of-thought-compress](skills/ctrlcot-dual-granularity-chain-of-thought-compress/SKILL.md) | Compress chain-of-thought reasoning using CtrlCoT's dual-granularity framework: hierarchical semantic abstraction combin... | 157 |
| [curate-train-refine-closed-loop-agentic-framework-](skills/curate-train-refine-closed-loop-agentic-framework/SKILL.md) | Build lightweight text classifiers from zero labeled data using an agentic Curate-Train-Refine loop. An LLM generates sy... | 148 |
| [d-orca-dialogue-centric-optimization-robust](skills/d-orca-dialogue-centric-optimization-robust/SKILL.md) | Build dialogue-centric audio-visual captioning pipelines that identify who spoke what and when in multi-party video conv... | 228 |
| [d2quant-accurate-low-bit-post-training-weight](skills/d2quant-accurate-low-bit-post-training-weight/SKILL.md) | Apply the D2Quant post-training weight quantization framework to compress LLMs to sub-4-bit precision (2-bit, 3-bit) wit... | 229 |
| [dart-diffusion-inspired-speculative-decoding](skills/dart-diffusion-inspired-speculative-decoding/SKILL.md) | Set up and use DART (Diffusion-Inspired Speculative Decoding) for fast LLM inference. DART replaces autoregressive draft... | 241 |
| [dart-ing-drift-dynamic-tracing](skills/dart-ing-drift-dynamic-tracing/SKILL.md) | Implement DART (Dynamic Attention-Guided Runtime Tracing) for inference-time FFN pruning in LLMs. Dynamically traces kno... | 225 |
| [darwin-dynamic-agentically-rewriting](skills/darwin-dynamic-agentically-rewriting/SKILL.md) | Evolutionary multi-agent code optimization using genetic algorithms. Agents mutate each other's training/configuration c... | 167 |
| [data-centric-interpretability-llm-based-multi-agen](skills/data-centric-interpretability-llm-based-multi-agen/SKILL.md) | Analyze LLM agent behavior across training runs using Sparse Autoencoder (SAE) features and LLM-summarizer pipelines. Gr... | 189 |
| [datachef-cooking-up-optimal](skills/datachef-cooking-up-optimal/SKILL.md) | Automate data recipe generation for LLM fine-tuning and adaptation. Generates executable data processing pipelines (filt... | 207 |
| [dcopilot-generative-ai-empowered-policy](skills/dcopilot-generative-ai-empowered-policy/SKILL.md) | Build hybrid LLM + hypernetwork systems that generate control policies for dynamic environments. Uses LLM-based reward s... | 297 |
| [decomposing-reasoning-efficiency](skills/decomposing-reasoning-efficiency/SKILL.md) | > | 248 |
| [decoupled-reasoning-implicit-fact](skills/decoupled-reasoning-implicit-fact/SKILL.md) | Build dual-model pipelines that decouple knowledge extraction from reasoning over long documents. Compress document chun... | 164 |
| [deepplanning-benchmarking-long-horizon-agentic](skills/deepplanning-benchmarking-long-horizon-agentic/SKILL.md) | Solve long-horizon planning tasks with verifiable constraints using the DeepPlanning methodology: proactive information ... | 155 |
| [deltaevolve-accelerating-scientific-discovery](skills/deltaevolve-accelerating-scientific-discovery/SKILL.md) | Iteratively evolve code solutions using momentum-driven semantic deltas instead of full-code histories. Use when: 'evolv... | 187 |
| [diffusion-pretrained-dense-contextual-embeddings](skills/diffusion-pretrained-dense-contextual-embeddings/SKILL.md) | Build production retrieval systems using pplx-embed, diffusion-pretrained dense and contextualized embedding models with... | 183 |
| [dispo-enhancing-training-efficiency](skills/dispo-enhancing-training-efficiency/SKILL.md) | Implement the DISPO reinforcement learning algorithm for training LLMs on mathematical reasoning with decoupled importan... | 277 |
| [distilling-reasoning-graph-concept](skills/distilling-reasoning-graph-concept/SKILL.md) | Distill LLM reasoning into a DAG of modular concept predictors for efficient, interpretable classification. Use when ask... | 170 |
| [dllm-agent-see-farther](skills/dllm-agent-see-farther/SKILL.md) | Design and implement multi-agent workflows using the DeepDiver hierarchical orchestration pattern with diffusion-inspire... | 163 |
| [do-reasoning-ask-questions](skills/do-reasoning-ask-questions/SKILL.md) | Information-theoretic question-asking framework for disambiguating user intent through structured yes/no questions. Uses... | 183 |
| [do-truly-benefit-longer](skills/do-truly-benefit-longer/SKILL.md) | Optimize LLM context length for post-editing and refinement pipelines. Applies research showing that naively adding docu... | 252 |
| [dr-kernel-reinforcement-learning-done](skills/dr-kernel-reinforcement-learning-done/SKILL.md) | Write high-performance Triton GPU kernels using Dr. Kernel's multi-turn refinement strategy: profile-guided optimization... | 200 |
| [drugr-optimizing-molecular-drugs](skills/drugr-optimizing-molecular-drugs/SKILL.md) | Optimize molecular drug candidates using LLM-based explicit pharmacological reasoning over SMILES structures. Applies th... | 187 |
| [dynamic-role-assignment-multi-agent](skills/dynamic-role-assignment-multi-agent/SKILL.md) | Dynamically assign specialized roles to multiple AI agents via a meta-debate protocol (proposal + peer review) before ru... | 182 |
| [dynaweb-model-based-reinforcement-learning](skills/dynaweb-model-based-reinforcement-learning/SKILL.md) | Build model-based RL training pipelines for web agents using learned world models (environment simulators) that predict ... | 193 |
| [e2pl-prompt-learning-incomplete](skills/e2pl-prompt-learning-incomplete/SKILL.md) | Design prompt-learning systems for incremental multi-view multi-label classification with missing data. Use when: 'handl... | 188 |
| [ecg-agent-on-device-tool-calling-agent](skills/ecg-agent-on-device-tool-calling-agent/SKILL.md) | Build on-device LLM tool-calling agents for multi-turn biomedical signal dialogue, following the ECG-Agent architecture.... | 296 |
| [edge-optimized-vision-language-underground-infrast](skills/edge-optimized-vision-language-underground-infrast/SKILL.md) | Build edge-deployable two-stage pipelines that combine lightweight segmentation with quantized Vision-Language Models fo... | 483 |
| [effgen-enabling-small-language](skills/effgen-enabling-small-language/SKILL.md) | Deploy and optimize small language models (SLMs) as autonomous agents using the effGen framework. Implements prompt comp... | 193 |
| [efficient-adaptable-detection-malicious](skills/efficient-adaptable-detection-malicious/SKILL.md) | > | 298 |
| [efficient-estimation-kernel-surrogate](skills/efficient-estimation-kernel-surrogate/SKILL.md) | Build kernel surrogate models to attribute how individual training tasks influence a target task's performance, capturin... | 199 |
| [efficient-table-retrieval-understanding](skills/efficient-table-retrieval-understanding/SKILL.md) | | | 178 |
| [emoshift-lightweight-activation-steering](skills/emoshift-lightweight-activation-steering/SKILL.md) | Implement lightweight activation steering for emotion-controllable speech synthesis. Adds learned steering vectors to LL... | 227 |
| [error-taxonomy-guided-prompt-optimization](skills/error-taxonomy-guided-prompt-optimization/SKILL.md) | | | 162 |
| [evaluating-kubernetes-performance-genai](skills/evaluating-kubernetes-performance-genai/SKILL.md) | Design and optimize Kubernetes-native GenAI inference platforms using Kueue job queuing, Dynamic Accelerator Slicer (DAS... | 217 |
| [evaluating-they-not-know](skills/evaluating-they-not-know/SKILL.md) | Build statistically efficient LLM evaluation pipelines that combine direct accuracy with pairwise comparison signals as ... | 185 |
| [event-vstream-event-driven-real-time-understanding](skills/event-vstream-event-driven-real-time-understanding/SKILL.md) | Build event-driven video stream processing pipelines that detect meaningful state transitions instead of processing ever... | 254 |
| [evocodebench-human-performance-benchmark-self-evol](skills/evocodebench-human-performance-benchmark-self-evol/SKILL.md) | Self-evolving code generation with iterative reflection and revision. Applies a feedback-driven loop where code is submi... | 174 |
| [evolve-evolutionary-search-llm-based](skills/evolve-evolutionary-search-llm-based/SKILL.md) | Evolutionary search framework for LLM-driven Verilog/RTL generation and PPA optimization. Uses MCTS for functional corre... | 174 |
| [evolving-tool-user-creator](skills/evolving-tool-user-creator/SKILL.md) | Transform Claude from a static tool user into a dynamic tool creator using the UCT (User-to-Creator Transformation) fram... | 181 |
| [experience-driven-multi-agent-systems-training-fre](skills/experience-driven-multi-agent-systems-training-fre/SKILL.md) | Build self-evolving multi-agent systems that accumulate tool-level expertise through structured interaction without mode... | 168 |
| [fast-slow-training-multimodal-visual](skills/fast-slow-training-multimodal-visual/SKILL.md) | Implement DualSpeed fast-slow training for multimodal LLMs with visual token pruning. Use when: 'speed up MLLM training'... | 230 |
| [fedkrso-communication-memory-federated](skills/fedkrso-communication-memory-federated/SKILL.md) | Implement FedKRSO (Federated K-Seed Random Subspace Optimization) for communication- and memory-efficient federated fine... | 214 |
| [fine-tuning-gpt-5-gpu-kernel](skills/fine-tuning-gpt-5-gpu-kernel/SKILL.md) | Generate optimized GPU kernels in Triton from PyTorch reference code using the Makora RL-based iterative refinement work... | 259 |
| [flashvid-video-training-free-tree-based](skills/flashvid-video-training-free-tree-based/SKILL.md) | Accelerate Video Large Language Models (VLLMs) by compressing visual tokens using FlashVID's training-free spatiotempora... | 168 |
| [focus-dllms-know-tame](skills/focus-dllms-know-tame/SKILL.md) | Deploy and optimize FOCUS inference for Diffusion Large Language Models (DLLMs). Configures attention-based token evicti... | 182 |
| [found-rl-foundation-model-enhanced-reinforcement](skills/found-rl-foundation-model-enhanced-reinforcement/SKILL.md) | Architect asynchronous VLM-enhanced RL training pipelines that decouple heavy foundation model inference from simulation... | 264 |
| [fraudshield-knowledge-graph-empowered](skills/fraudshield-knowledge-graph-empowered/SKILL.md) | Detect and defend against fraudulent content in LLM inputs using knowledge-graph-augmented analysis. Builds a fraud tact... | 269 |
| [from-pragmas-partners-symbiotic](skills/from-pragmas-partners-symbiotic/SKILL.md) | Agentic High-Level Synthesis (HLS) optimization: autonomously analyze, insert, and tune C/C++ HLS pragmas (pipeline, unr... | 186 |
| [from-sparse-decisions-dense](skills/from-sparse-decisions-dense/SKILL.md) | Build content moderation and safety classification systems using multi-attribute trajectory reasoning instead of binary ... | 261 |
| [from-utterance-vividity-training](skills/from-utterance-vividity-training/SKILL.md) | Train expressive subtitle translation LLMs using Adaptive Local Preference Optimization (ALPO) — a segment-level prefere... | 257 |
| [frost-filtering-reasoning-outliers](skills/frost-filtering-reasoning-outliers/SKILL.md) | Implement FROST (Filtering Reasoning Outliers with Attention) to prune unnecessary reasoning steps from LLM chain-of-tho... | 252 |
| [gametalk-training-strategic-conversation](skills/gametalk-training-strategic-conversation/SKILL.md) | Build multi-agent strategic conversation systems where LLMs negotiate, coordinate, and optimize long-term objectives thr... | 207 |
| [gflowpo-generative-flow-network](skills/gflowpo-generative-flow-network/SKILL.md) | Optimize LLM prompts using GFlowPO's iterative generate-evaluate-refine loop with diversity-preserving exploration and d... | 171 |
| [greprag-empirical-study-optimization](skills/greprag-empirical-study-optimization/SKILL.md) | Lightweight, index-free repository-level code retrieval using ripgrep for context-aware code completion. Uses LLM-genera... | 190 |
| [group-distributionally-robust-optimization-driven](skills/group-distributionally-robust-optimization-driven/SKILL.md) | Apply Group Distributionally Robust Optimization (GDRO) to RL-based LLM training. Dynamically classify prompts by diffic... | 205 |
| [he-snr-uncovering-latent-logic](skills/he-snr-uncovering-latent-logic/SKILL.md) | Evaluate and optimize LLM training data quality for software engineering tasks using the HE-SNR (High-Entropy Signal-to-... | 234 |
| [high-fidelity-textual-user](skills/high-fidelity-textual-user/SKILL.md) | Build RL-optimized unified textual user representations from heterogeneous data sources (profiles, activity logs, search... | 227 |
| [hqp-sensitivity-aware-hybrid-quantization](skills/hqp-sensitivity-aware-hybrid-quantization/SKILL.md) | Apply the HQP framework to compress and accelerate PyTorch models for edge deployment using sensitivity-aware structural... | 189 |
| [hylra-hybrid-layer-reuse](skills/hylra-hybrid-layer-reuse/SKILL.md) | Implement HyLRA (Hybrid Layer Reuse Attention) for efficient long-context LLM inference. Profiles layer-wise sparsity, a... | 190 |
| [hyperoffload-graph-driven-hierarchical-memory](skills/hyperoffload-graph-driven-hierarchical-memory/SKILL.md) | Design and implement compiler-driven hierarchical memory offloading for LLM inference and training on multi-tier memory ... | 226 |
| [ide-bench-evaluating-as-ide](skills/ide-bench-evaluating-as-ide/SKILL.md) | Apply IDE-Bench's structured agent workflow for tackling real-world software engineering tasks: systematic exploration b... | 149 |
| [improve-systems-user-logs](skills/improve-systems-user-logs/SKILL.md) | Implement the UNO (User log-driveN Optimization) framework to improve LLM-powered systems by distilling user interaction... | 201 |
| [innovator-vl-multimodal-scientific-discovery](skills/innovator-vl-multimodal-scientific-discovery/SKILL.md) | Build data-efficient multimodal scientific ML pipelines using Innovator-VL's principled training methodology. Applies tr... | 247 |
| [internalizing-multi-agent-reasoning-accurate](skills/internalizing-multi-agent-reasoning-accurate/SKILL.md) | Distill multi-agent reasoning into a single efficient model for recommendation or retrieval. Use when: 'build a recommen... | 174 |
| [internalizing-reasoning-discovery-replay](skills/internalizing-reasoning-discovery-replay/SKILL.md) | Apply the STIR (Self-Distilled Tools for Internal Reasoning) pattern to build systems that discover reusable reasoning p... | 253 |
| [intraslice-high-performance-structural-pruning](skills/intraslice-high-performance-structural-pruning/SKILL.md) | Implement IntraSlice block-intra PCA structural pruning for LLMs. Compresses Transformer attention and FFN modules by ap... | 204 |
| [jade-bridging-strategic-operational-gap](skills/jade-bridging-strategic-operational-gap/SKILL.md) | Build jointly-optimized agentic RAG pipelines using the JADE pattern: a central planner co-adapted with specialized exec... | 248 |
| [jetformer-scalable-transformer-jet](skills/jetformer-scalable-transformer-jet/SKILL.md) | Build, train, compress, and deploy JetFormer encoder-only Transformers for particle jet tagging -- from offline analysis... | 167 |
| [joint-continual-learning-local](skills/joint-continual-learning-local/SKILL.md) | Implement DA-GRPO (Dual-Advantage Group Relative Policy Optimization) for jointly training local small language models w... | 270 |
| [just-in-time-reinforcement-learning-continual](skills/just-in-time-reinforcement-learning-continual/SKILL.md) | Implement JitRL-style continual learning for LLM agents: training-free policy optimization via experience memory, advant... | 203 |
| [knowledge-restoration-driven-prompt-optimization](skills/knowledge-restoration-driven-prompt-optimization/SKILL.md) | | | 211 |
| [kv-core-benchmarking-data-dependent-low-rank](skills/kv-core-benchmarking-data-dependent-low-rank/SKILL.md) | Benchmark and analyze KV-cache low-rank compressibility in LLMs using SVD-based evaluation and Normalized Effective Rank... | 195 |
| [large-model-powered-evolutionary-code](skills/large-model-powered-evolutionary-code/SKILL.md) | Iteratively optimize code performance using LLM-driven evolutionary search on a phylogenetic tree. Applies PhyloEvolve-s... | 186 |
| [latentchem-textual-cot-latent](skills/latentchem-textual-cot-latent/SKILL.md) | Apply LatentChem's latent-space reasoning paradigm to chemical computation tasks -- replacing verbose textual Chain-of-T... | 189 |
| [layer-wise-lora-fine-tuning-similarity](skills/layer-wise-lora-fine-tuning-similarity/SKILL.md) | Selectively apply LoRA adapters to only the most important transformer layers using CKA similarity-based layer importanc... | 221 |
| [learning-rate-matters-vanilla](skills/learning-rate-matters-vanilla/SKILL.md) | Configure optimal learning rates for LoRA fine-tuning of LLMs. Generates hyperparameter search configs, training scripts... | 203 |
| [legalone-family-foundation-reliable](skills/legalone-family-foundation-reliable/SKILL.md) | Build domain-specialized LLM training pipelines using the LegalOne three-phase methodology: Plasticity-Adjusted Sampling... | 259 |
| [lemon-agent-technical-report](skills/lemon-agent-technical-report/SKILL.md) | Orchestrate multi-agent workflows using the Lemon Agent orchestrator-worker pattern with hierarchical scheduling, progre... | 186 |
| [less-enough-synthesizing-diverse](skills/less-enough-synthesizing-diverse/SKILL.md) | Synthesize maximally diverse training data for LLM post-training using Feature Activation Coverage (FAC). Identifies mis... | 181 |
| [leveraging-turkish-skill-extraction](skills/leveraging-turkish-skill-extraction/SKILL.md) | Extract and normalize skills from job postings using a two-stage LLM pipeline: dynamic few-shot skill identification fol... | 198 |
| [linear-merging-unlocks-simple](skills/linear-merging-unlocks-simple/SKILL.md) | Use linear model merging as a cheap proxy for data mixture optimization (DMO) in multimodal LLM fine-tuning. Instead of ... | 168 |
| [llamea-sage-guiding-automated-algorithm](skills/llamea-sage-guiding-automated-algorithm/SKILL.md) | Guide LLM-driven algorithm generation using AST structural feedback and explainable AI. Extracts graph-theoretic and com... | 226 |
| [llm-assisted-logic-rule-learning](skills/llm-assisted-logic-rule-learning/SKILL.md) | Build deterministic, interpretable anomaly detection rule sets for time series data using LLM-driven labeling, symbolic ... | 181 |
| [llm-autodp-automatic-data-processing](skills/llm-autodp-automatic-data-processing/SKILL.md) | Automatically generate and optimize data processing pipelines for LLM fine-tuning datasets using an agent-driven iterati... | 168 |
| [llm-enhanced-reinforcement-learning-long-term](skills/llm-enhanced-reinforcement-learning-long-term/SKILL.md) | Build hierarchical recommendation systems that combine LLM semantic planning with RL fine-grained optimization for long-... | 247 |
| [llm-prompt-evaluation-educational](skills/llm-prompt-evaluation-educational/SKILL.md) | Systematically design, evaluate, and rank LLM prompts for educational applications using tournament-style Glicko-2 compa... | 223 |
| [llms-encode-failures-predicting](skills/llms-encode-failures-predicting/SKILL.md) | Build probe-based routing systems that predict LLM success before generation and route queries to cost-optimal models. U... | 192 |
| [lycheedecode-accelerating-long-context-inference](skills/lycheedecode-accelerating-long-context-inference/SKILL.md) | Accelerate long-context LLM inference using hybrid-head sparse decoding with HardKuma-based head partitioning. Implement... | 179 |
| [made-benchmark-environments-closed-loop](skills/made-benchmark-environments-closed-loop/SKILL.md) | Build closed-loop discovery benchmarks where an agent iteratively proposes, evaluates, and refines candidates under a fi... | 144 |
| [magellan-autonomous-discovery-compiler](skills/magellan-autonomous-discovery-compiler/SKILL.md) | Evolve compiler optimization heuristics by coupling LLM code generation with evolutionary search and autotuning. Synthes... | 158 |
| [malloc-benchmarking-memory-aware-long](skills/malloc-benchmarking-memory-aware-long/SKILL.md) | Apply memory-aware compression strategies to long-sequence recommendation systems. Benchmark KV-cache compression techni... | 248 |
| [marble-multi-agent-reasoning-bioinformatics](skills/marble-multi-agent-reasoning-bioinformatics/SKILL.md) | Iteratively refine bioinformatics and ML models using MARBLE's multi-agent debate framework with role-specialized agents... | 206 |
| [markovscale-optimal-sequential-scaling](skills/markovscale-optimal-sequential-scaling/SKILL.md) | Implement MarkovScale's principled sequential scaling for LLM inference pipelines. Models retry/refinement loops as a tw... | 193 |
| [martingale-foresight-sampling-principled](skills/martingale-foresight-sampling-principled/SKILL.md) | Implement Martingale Foresight Sampling (MFS) for principled LLM decoding with lookahead search. Replaces heuristic beam... | 220 |
| [mascot-multi-agent-socio-collaborative-companion](skills/mascot-multi-agent-socio-collaborative-companion/SKILL.md) | Design and orchestrate multi-agent companion systems where each agent maintains a distinct persona and contributes diver... | 244 |
| [mata-multiagent-framework-for](skills/mata-multiagent-framework-for/SKILL.md) | Multi-agent table question answering using MATA's three-path reasoning strategy (Chain-of-Thought, Program-of-Thought, T... | 167 |
| [memadapter-fast-alignment-across](skills/memadapter-fast-alignment-across/SKILL.md) | Unify heterogeneous agent memory systems (explicit graphs, parametric weights, latent KV-caches) via generative subgraph... | 166 |
| [mirror-multi-agent-framework-iterative](skills/mirror-multi-agent-framework-iterative/SKILL.md) | Translate natural language optimization problems into mathematical models and solver code using MIRROR's multi-agent pip... | 166 |
| [mmr-bench-comprehensive-benchmark-multimodal](skills/mmr-bench-comprehensive-benchmark-multimodal/SKILL.md) | Build cost-aware multimodal LLM routing systems that select the best model per query based on input signals, budget cons... | 175 |
| [more-than-quick-glance](skills/more-than-quick-glance/SKILL.md) | Implement LASER-KV-style KV-cache compression for LLM inference pipelines using block-wise accumulative budgeting and hy... | 256 |
| [mrag-benchmarking-retrieval-augmented-generation](skills/mrag-benchmarking-retrieval-augmented-generation/SKILL.md) | Build and evaluate biomedical RAG pipelines using the MRAG benchmark methodology. Configures retrieval, prompting, and g... | 183 |
| [muco-multi-turn-contrastive-learning](skills/muco-multi-turn-contrastive-learning/SKILL.md) | Implement multi-turn contrastive learning for multimodal embedding models. Restructures query-target pairs as multi-turn... | 260 |
| [multi-agent-teams-hold-experts](skills/multi-agent-teams-hold-experts/SKILL.md) | Prevent expertise dilution in multi-agent LLM workflows by applying findings from 'Multi-Agent Teams Hold Experts Back' ... | 151 |
| [multi-agentic-ai-fairness-aware-accelerated](skills/multi-agentic-ai-fairness-aware-accelerated/SKILL.md) | Design and implement multi-agent systems for fairness-aware, low-latency inference orchestration across distributed edge... | 200 |
| [multi-field-tool-retrieval](skills/multi-field-tool-retrieval/SKILL.md) | Implement multi-field tool retrieval systems that decompose tool documentation into structured fields (description, para... | 284 |
| [no-global-plan-chain-of-thought](skills/no-global-plan-chain-of-thought/SKILL.md) | Optimize Chain-of-Thought reasoning by detecting when CoT can be bypassed and identifying pivot positions that capture p... | 196 |
| [nwa-mending-spatial-integrity-torn](skills/nwa-mending-spatial-integrity-torn/SKILL.md) | Implement spatially-aware vision token pruning for VLMs using the Nüwa two-stage framework: separation, alignment, and a... | 234 |
| [omnirag-agent-agentic-omnimodal-reasoning](skills/omnirag-agent-agentic-omnimodal-reasoning/SKILL.md) | Build agentic multimodal RAG pipelines that answer questions over long audio-video content under resource constraints. U... | 278 |
| [on-impact-agentsmd-files](skills/on-impact-agentsmd-files/SKILL.md) | Generate and optimize AGENTS.md / CLAUDE.md repository instruction files to reduce AI coding agent runtime and token con... | 196 |
| [on-uncertainty-model-based-multi-agent](skills/on-uncertainty-model-based-multi-agent/SKILL.md) | Apply entropy-based uncertainty analysis to multi-agent LLM systems. Diagnose when multi-agent setups hurt performance, ... | 184 |
| [one-size-many-fits](skills/one-size-many-fits/SKILL.md) | Build group-aware advertising image generation systems that align diverse user-segment click preferences instead of opti... | 236 |
| [optimal-turkish-subword-strategies](skills/optimal-turkish-subword-strategies/SKILL.md) | Design and evaluate subword tokenizers for Turkish and other morphologically rich languages (MRLs) using the vocabulary-... | 253 |
| [optimizing-prompts-causal-approach](skills/optimizing-prompts-causal-approach/SKILL.md) | Optimize LLM prompts using causal inference (CPO). Isolates true prompt effectiveness from query difficulty via Double M... | 156 |
| [optimizing-small-sample-experience-learning-llm-ba](skills/optimizing-small-sample-experience-learning-llm-ba/SKILL.md) | Implement the ExperienceWeaver hierarchical experience-learning framework to improve text quality from small feedback se... | 195 |
| [out-memory-barrier-highly](skills/out-memory-barrier-highly/SKILL.md) | Configure and run OOMB for memory-efficient long-context LLM training with million-token sequences on limited GPUs. Trig... | 200 |
| [pabu-progress-aware-belief-update](skills/pabu-progress-aware-belief-update/SKILL.md) | Apply Progress-Aware Belief Update (PABU) to build efficient LLM agents that track task progress and selectively retain ... | 210 |
| [pand-prompt-aware-neighborhood-distillation](skills/pand-prompt-aware-neighborhood-distillation/SKILL.md) | Implement PAND (Prompt-Aware Neighborhood Distillation) for distilling Vision-Language Models into lightweight networks ... | 189 |
| [parameter-efficient-multi-task-fine-tuning-code-re](skills/parameter-efficient-multi-task-fine-tuning-code-re/SKILL.md) | Configure and execute multi-task QLoRA fine-tuning of code models for code generation, translation, and summarization. U... | 223 |
| [parse-open-domain-reasoning-question](skills/parse-open-domain-reasoning-question/SKILL.md) | Build and evaluate reasoning-focused QA systems for low-resource languages using the PARSE methodology: structured promp... | 220 |
| [pathwise-planning-world-automated](skills/pathwise-planning-world-automated/SKILL.md) | Multi-agent heuristic design framework that uses an entailment graph, policy/world-model/critic agents, and routed refle... | 165 |
| [pcl-reasoner-v15-advancing-math-reasoning-offline](skills/pcl-reasoner-v15-advancing-math-reasoning-offline/SKILL.md) | Implement offline reinforcement learning pipelines for LLM reasoning tasks — decoupling data collection from training fo... | 255 |
| [pearl-prototype-enhanced-alignment-label-efficient](skills/pearl-prototype-enhanced-alignment-label-efficient/SKILL.md) | | | 222 |
| [persodpo-scalable-preference-optimization](skills/persodpo-scalable-preference-optimization/SKILL.md) | Build scalable preference optimization pipelines for persona-grounded dialogue systems using multi-LLM evaluation. Use w... | 173 |
| [personality-as-relational-infrastructure](skills/personality-as-relational-infrastructure/SKILL.md) | Design LLM messaging systems that infuse Big Five personality traits for sustained user engagement. Uses aggregate-expos... | 252 |
| [phenolip-integrating-phenotype-ontology](skills/phenolip-integrating-phenotype-ontology/SKILL.md) | Build phenotype-aware medical vision-language models by integrating ontology knowledge graphs into CLIP-style pretrainin... | 234 |
| [pop-prefill-only-pruning-inference](skills/pop-prefill-only-pruning-inference/SKILL.md) | | | 193 |
| [predicting-improving-test-time-scaling](skills/predicting-improving-test-time-scaling/SKILL.md) | Implement Scaling-Law Guided (SLG) Search for test-time compute optimization. Uses reward tail distribution estimation (... | 209 |
| [promptrl-prompt-matters-rl](skills/promptrl-prompt-matters-rl/SKILL.md) | Implement PromptRL-style joint prompt-refinement + RL training loops for flow-based image generation. Use when the user ... | 196 |
| [proopf-benchmarking-improving-professional-grade](skills/proopf-benchmarking-improving-professional-grade/SKILL.md) | Translate natural-language power system operational requirements into executable Optimal Power Flow (OPF) optimization c... | 218 |
| [protean-compiler-agile-framework](skills/protean-compiler-agile-framework/SKILL.md) | Guide fine-grained LLVM compiler phase ordering using the Protean framework's agile optimization approach — clustering p... | 149 |
| [pruning-minimal-reasoning-graphs](skills/pruning-minimal-reasoning-graphs/SKILL.md) | > | 154 |
| [quasar-universal-autonomous-system](skills/quasar-universal-autonomous-system/SKILL.md) | Build autonomous multi-scale scientific simulation pipelines using the QUASAR architecture: a Strategist-Operator-Evalua... | 165 |
| [query-efficient-agentic-graph-extraction](skills/query-efficient-agentic-graph-extraction/SKILL.md) | > | 239 |
| [r1-syntheticvl-synthetic-data-generative](skills/r1-syntheticvl-synthetic-data-generative/SKILL.md) | Synthesize high-quality multimodal training data using Collective Adversarial Data Synthesis (CADS). Implements a cyclic... | 226 |
| [ragturk-best-practices-retrieval](skills/ragturk-best-practices-retrieval/SKILL.md) | Design and optimize RAG pipelines for Turkish and other morphologically rich languages (Turkish, Finnish, Hungarian, Kor... | 208 |
| [ral-bench-benchmarking-application-level-functiona](skills/ral-bench-benchmarking-application-level-functiona/SKILL.md) | Generate and evaluate complete multi-file application repositories with both functional correctness and non-functional q... | 179 |
| [rapid-real-time-deterministic-trajectory](skills/rapid-real-time-deterministic-trajectory/SKILL.md) | Distill diffusion-based trajectory planners into fast deterministic policies using score-regularized optimization and sa... | 185 |
| [rapo-risk-aware-preference-optimization](skills/rapo-risk-aware-preference-optimization/SKILL.md) | Apply risk-aware preference optimization to make LLM reasoning chains safer against jailbreak attacks. Implements adapti... | 203 |
| [rc-grpo-reward-conditioned-group-relative](skills/rc-grpo-reward-conditioned-group-relative/SKILL.md) | Implement reward-conditioned training pipelines for multi-turn tool-calling agents using RC-GRPO. Injects discrete rewar... | 228 |
| [read-as-human-compressing](skills/read-as-human-compressing/SKILL.md) | Compress long contexts using the RAM (Read As Human) strategy: partition text into segments, score relevance against a q... | 242 |
| [reasoning-augmented-representations-multimodal-ret](skills/reasoning-augmented-representations-multimodal-ret/SKILL.md) | Decouple reasoning from embedding compression in multimodal retrieval pipelines by enriching queries and corpus entries ... | 223 |
| [rebel-hidden-knowledge-recovery](skills/rebel-hidden-knowledge-recovery/SKILL.md) | Machine unlearning for LLMs aims to remove sensitive or copyrighted data from trained models. Implements techniques from... | 28 |
| [regular-variational-latent-reasoning](skills/regular-variational-latent-reasoning/SKILL.md) | Compress verbose chain-of-thought reasoning into compact latent state representations guided by rendered visual summarie... | 236 |
| [reinforced-attention-learning](skills/reinforced-attention-learning/SKILL.md) | Implement Reinforced Attention Learning (RAL) for multimodal LLMs — optimize attention distributions via policy gradient... | 216 |
| [reinforcement-learning-self-distillation](skills/reinforcement-learning-self-distillation/SKILL.md) | Implement Self-Distillation Policy Optimization (SDPO) for RL training loops that convert rich textual feedback into den... | 252 |
| [remedit-diffusion-editing-riemannian](skills/remedit-diffusion-editing-riemannian/SKILL.md) | Implement Riemannian-geometry-based diffusion image editing pipelines using geodesic latent navigation, dual-SLERP blend... | 244 |
| [report-nsf-workshop-ai](skills/report-nsf-workshop-ai/SKILL.md) | Apply AI techniques from the NSF AI-for-EDA workshop to hardware design tasks: RTL code generation from natural language... | 276 |
| [reprompt-prompt-generation-intelligent](skills/reprompt-prompt-generation-intelligent/SKILL.md) | Generate optimized system and user prompts for coding agents using requirements engineering principles from the REprompt... | 191 |
| [residual-context-diffusion](skills/residual-context-diffusion/SKILL.md) | Implement and apply Residual Context Diffusion (RCD) for diffusion language models. Converts wasted computation from rem... | 193 |
| [rethinking-generative-recommender-tokenizer](skills/rethinking-generative-recommender-tokenizer/SKILL.md) | Build recommendation-native Semantic ID tokenizers using the ReSID framework (Field-Aware Masked Auto-Encoding + Globall... | 154 |
| [rethinking-genomic-modeling-optical](skills/rethinking-genomic-modeling-optical/SKILL.md) | Implement OpticalDNA-style pipelines that render DNA sequences as 2D visual layouts and process them with OCR-capable vi... | 248 |
| [rethinking-irregular-time-series](skills/rethinking-irregular-time-series/SKILL.md) | Design and implement irregular time series classification pipelines for clinical/ICU data with high missing-value rates.... | 186 |
| [rethinking-role-entropy-optimizing](skills/rethinking-role-entropy-optimizing/SKILL.md) | Optimize LLM agent tool-use behavior using entropy reduction as a quality signal. Reduces excessive tool calls and impro... | 175 |
| [rethinking-trust-region-reinforcement](skills/rethinking-trust-region-reinforcement/SKILL.md) | Implement Divergence Proximal Policy Optimization (DPPO) for LLM reinforcement learning fine-tuning, replacing PPO's rat... | 204 |
| [rethinking-value-agent-generated-tests](skills/rethinking-value-agent-generated-tests/SKILL.md) | Optimize agent test-writing strategy for issue resolution by reallocating interaction budget from excessive test generat... | 163 |
| [revisiting-adaptive-rounding-vectorized](skills/revisiting-adaptive-rounding-vectorized/SKILL.md) | Implement VQRound -- a parameter-efficient adaptive rounding framework for LLM post-training quantization that reparamet... | 175 |
| [rewards-as-labels-revisiting](skills/rewards-as-labels-revisiting/SKILL.md) | Implement the REAL (Rewards as Labels) framework for LLM reinforcement learning, which reformulates RLVR policy optimiza... | 218 |
| [risk-awareness-injection-calibrating](skills/risk-awareness-injection-calibrating/SKILL.md) | Implement Risk Awareness Injection (RAI) to defend vision-language models against multimodal jailbreak attacks without r... | 244 |
| [roma-recursive-open-meta-agent](skills/roma-recursive-open-meta-agent/SKILL.md) | Decompose long-horizon, multi-step tasks using ROMA's recursive meta-agent pattern: Atomizer decides if a task needs spl... | 185 |
| [rpo-rag-aligning-small-relation-aware](skills/rpo-rag-aligning-small-relation-aware/SKILL.md) | Build knowledge-graph-grounded RAG pipelines that align small LLMs (under 8B params) with relation-aware preference opti... | 259 |
| [ruleflow-generating-reusable-program](skills/ruleflow-generating-reusable-program/SKILL.md) | Optimize Pandas code by discovering per-program improvements, generalizing them into reusable rewrite rules, and applyin... | 160 |
| [rulesmith-multi-agent-automated-game](skills/rulesmith-multi-agent-automated-game/SKILL.md) | Automated game balancing using multi-agent LLM self-play coupled with Bayesian optimization. Use when the user asks to '... | 193 |
| [rvb-automating-ai-system](skills/rvb-automating-ai-system/SKILL.md) | Harden code and AI guardrails through iterative Red Team vs Blue Team adversarial games. Use when the user says 'harden ... | 222 |
| [rvcbench-benchmarking-robustness-voice](skills/rvcbench-benchmarking-robustness-voice/SKILL.md) | Benchmark and harden voice cloning systems against real-world degradation using the RVCBench framework. Evaluates VC mod... | 164 |
| [s3-cot-self-sampled-succinct-reasoning](skills/s3-cot-self-sampled-succinct-reasoning/SKILL.md) | Apply dual-cognitive reasoning (System 1 fast / System 2 slow) to compress verbose chain-of-thought into succinct, effic... | 201 |
| [scalable-generative-game-engine](skills/scalable-generative-game-engine/SKILL.md) | Design and deploy real-time generative game engines that break the Memory Wall via hardware-algorithm co-design. Covers ... | 162 |
| [scaled-surrogate-gradient-codec-aware-learning](skills/scaled-surrogate-gradient-codec-aware-learning/SKILL.md) | Build end-to-end video processing pipelines that train learned downsamplers/upsamplers through real non-differentiable c... | 215 |
| [self-evolving-recommendation-system-end-to-end](skills/self-evolving-recommendation-system-end-to-end/SKILL.md) | Build autonomous ML optimization pipelines that use LLM agents to generate, evaluate, and deploy model improvements in a... | 168 |
| [sere-similarity-based-expert-re-routing](skills/sere-similarity-based-expert-re-routing/SKILL.md) | Deploy SERE (Similarity-based Expert Re-routing) to accelerate MoE model batch decoding in vLLM by dynamically skipping ... | 223 |
| [shine-scalable-in-context-hypernetwork](skills/shine-scalable-in-context-hypernetwork/SKILL.md) | Guide Claude to apply SHINE's single-pass context-to-LoRA hypernetwork technique for converting document knowledge into ... | 225 |
| [skillrl-evolving-agents-recursive](skills/skillrl-evolving-agents-recursive/SKILL.md) | Build self-improving agent systems that distill raw execution traces into a hierarchical skill library (SkillBank) and r... | 184 |
| [small-beautiful-practical-log](skills/small-beautiful-practical-log/SKILL.md) | Build efficient log parsing systems that extract structured templates from raw log messages using a dual-cache architect... | 191 |
| [snapmla-efficient-longcontext-mla](skills/snapmla-efficient-longcontext-mla/SKILL.md) | While FP8 attention has shown substantial promise in innovations like FlashAttention-3, its integration into the decodin... | 88 |
| [snapmla-long-context-mla-decoding](skills/snapmla-long-context-mla-decoding/SKILL.md) | Deploy and optimize FP8-quantized Multi-head Latent Attention (MLA) decoding for long-context LLM inference on Hopper GP... | 186 |
| [sparse-sparse-safety-unsafe](skills/sparse-sparse-safety-unsafe/SKILL.md) | Audit and harden Mixture-of-Experts (MoE) LLM deployments against unsafe routing vulnerabilities using RoSais scoring an... | 184 |
| [sparseeval-evaluation-sparse-optimization](skills/sparseeval-evaluation-sparse-optimization/SKILL.md) | Efficiently evaluate LLMs on benchmarks by selecting a small subset of anchor items via sparse optimization, reproducing... | 221 |
| [spava-accelerating-long-video-understanding](skills/spava-accelerating-long-video-understanding/SKILL.md) | Implement Spava-style sequence-parallel approximate attention for accelerating long-video inference across multiple GPUs... | 200 |
| [spell-synthesis-programmatic-edits](skills/spell-synthesis-programmatic-edits/SKILL.md) | Automate library migrations by synthesizing reusable code transformation scripts. Uses LLM-generated migration examples ... | 209 |
| [star-similarity-guided-teacher-assisted-refinement](skills/star-similarity-guided-teacher-assisted-refinement/SKILL.md) | Distill function-calling capabilities from large language models into super-tiny models (0.6B-4B) using the STAR framewo... | 281 |
| [step-35-flash-open](skills/step-35-flash-open/SKILL.md) | Build efficient agentic AI systems using sparse MoE routing, hybrid sliding-window/full attention, multi-token predictio... | 226 |
| [streaming-dllm-accelerating-diffusion-suffix](skills/streaming-dllm-accelerating-diffusion-suffix/SKILL.md) | Accelerate diffusion LLM inference via suffix pruning and dynamic confidence-aware decoding. Use when: 'speed up diffusi... | 238 |
| [swe-pruner-self-adaptive-context-pruning](skills/swe-pruner-self-adaptive-context-pruning/SKILL.md) | | | 186 |
| [swe-replay-test-time-scaling-software](skills/swe-replay-test-time-scaling-software/SKILL.md) | Efficient test-time scaling for software engineering agents using trajectory recycling and explore-exploit branching (SW... | 158 |
| [t-llm-teaching-forecast-time](skills/t-llm-teaching-forecast-time/SKILL.md) | Implement temporal distillation pipelines that teach LLMs to forecast time series by training a lightweight trend+freque... | 155 |
| [teaching-evaluating-reason-about](skills/teaching-evaluating-reason-about/SKILL.md) | Apply knowledge-augmented reasoning distillation for polymer design tasks. Builds structured Chain-of-Thought pipelines ... | 202 |
| [team-temporal-spatial-consistency-guided](skills/team-temporal-spatial-consistency-guided/SKILL.md) | Accelerate MoE diffusion language model inference using TEAM's temporal-spatial consistency framework. Implements expert... | 193 |
| [ternarylm-memory-efficient-modeling-native](skills/ternarylm-memory-efficient-modeling-native/SKILL.md) | Implement native 1-bit ternary quantization {-1, 0, +1} for training memory-efficient language models from scratch. Cove... | 230 |
| [text-summarization-global-structure](skills/text-summarization-global-structure/SKILL.md) | Summarize long documents while preserving global semantic structure and logical coherence using topology-guided pruning ... | 165 |
| [thinking-broad-acting-fast](skills/thinking-broad-acting-fast/SKILL.md) | Build production search relevance systems using Multi-Perspective Chain-of-Thought distillation into lightweight student... | 210 |
| [timely-machine-awareness-time](skills/timely-machine-awareness-time/SKILL.md) | Apply time-budget-aware reasoning to agentic tasks with tool calls. Dynamically adjust strategy depth, tool call frequen... | 171 |
| [tokenomics-quantifying-where-tokens](skills/tokenomics-quantifying-where-tokens/SKILL.md) | Analyze and optimize token consumption in LLM-based multi-agent software engineering workflows. Maps agent execution tra... | 227 |
| [towards-automated-kernel-generation](skills/towards-automated-kernel-generation/SKILL.md) | Automate GPU kernel generation and optimization using LLM-driven agentic workflows with profiling feedback loops. Use wh... | 159 |
| [towards-green-ai-decoding](skills/towards-green-ai-decoding/SKILL.md) | Optimize LLM-generated code for energy efficiency by detecting and suppressing babbling behavior (excess tokens like red... | 229 |
| [towards-holographic-characteristic-short-text](skills/towards-holographic-characteristic-short-text/SKILL.md) | Apply the Holographic Characteristic of LLMs to generate efficient short text by extracting keywords early then completi... | 150 |
| [towards-sample-efficient-stable-reinforcement](skills/towards-sample-efficient-stable-reinforcement/SKILL.md) | | | 191 |
| [towards-understanding-best-practices](skills/towards-understanding-best-practices/SKILL.md) | Quantize vision-language models (VLMs) component-by-component using optimal bit-width strategies derived from multimodal... | 189 |
| [tracenas-zero-shot-pruning-gradient](skills/tracenas-zero-shot-pruning-gradient/SKILL.md) | Implement TraceNAS-style zero-shot LLM structured pruning using gradient trace correlation as a scale-invariant proxy. J... | 226 |
| [training-data-selection-gradient](skills/training-data-selection-gradient/SKILL.md) | Implement Orthogonal Gradient Selection (OGS) for efficient domain adaptation of LLMs—select training data whose gradien... | 196 |
| [training-multi-turn-search-agent](skills/training-multi-turn-search-agent/SKILL.md) | Build and train multi-turn search agents using BranPO (Branching Relative Policy Optimization) with contrastive dynamic ... | 177 |
| [understanding-agent-scaling-llm-based](skills/understanding-agent-scaling-llm-based/SKILL.md) | Design diversity-aware multi-agent systems that maximize performance with fewer agents. Uses information-theoretic K* ef... | 204 |
| [unicog-uncovering-cognitive-abilities](skills/unicog-uncovering-cognitive-abilities/SKILL.md) | Analyze and diagnose LLM reasoning through latent cognitive ability decomposition inspired by the UniCog framework. Deco... | 178 |
| [unicomp-unified-evaluation-compression](skills/unicomp-unified-evaluation-compression/SKILL.md) | Guide Claude through evaluating and recommending LLM compression strategies (pruning, quantization, distillation) using ... | 177 |
| [unleashing-potential-sparse-attention](skills/unleashing-potential-sparse-attention/SKILL.md) | Implement SparseCTR's three-branch sparse attention for efficient CTR prediction on long user behavior sequences. Use wh... | 321 |
| [usage-effects-requirements-ai-coding](skills/usage-effects-requirements-ai-coding/SKILL.md) | Optimize AI coding assistant interactions using empirical enterprise findings on usage patterns, productivity factors, a... | 228 |
| [use-graph-it-needs](skills/use-graph-it-needs/SKILL.md) | Implement adaptive RAG pipelines that route queries to dense retrieval, graph-based retrieval, or a weighted fusion base... | 254 |
| [v0-generalist-value-any-policy](skills/v0-generalist-value-any-policy/SKILL.md) | Implement V0-style generalist value estimation that profiles any LLM policy from behavioral history rather than paramete... | 169 |
| [veq-modality-adaptive-quantization-moe](skills/veq-modality-adaptive-quantization-moe/SKILL.md) | Apply VEQ modality-adaptive quantization to compress MoE Vision-Language Models with minimal accuracy loss. Implements d... | 199 |
| [vespo-variational-sequence-level-soft](skills/vespo-variational-sequence-level-soft/SKILL.md) | Implement VESPO (Variational Sequence-Level Soft Policy Optimization) for stable off-policy LLM reinforcement learning. ... | 189 |
| [vica-multimodal-vision-only-cross-attention](skills/vica-multimodal-vision-only-cross-attention/SKILL.md) | Implement and apply the ViCA (Vision-only Cross-Attention) architecture to reduce visual computation in multimodal LLMs ... | 232 |
| [vidvec-unlocking-video-mllm](skills/vidvec-unlocking-video-mllm/SKILL.md) | Extract high-quality video-text embeddings from generative MLLMs using intermediate-layer representations and text-only ... | 197 |
| [viola-video-in-context-learning](skills/viola-video-in-context-learning/SKILL.md) | Apply the VIOLA framework for label-efficient in-context learning on video or multimodal data. Uses density-uncertainty-... | 221 |
| [visiontrim-unified-vision-token](skills/visiontrim-unified-vision-token/SKILL.md) | Implement VisionTrim's training-free visual token compression for multimodal LLMs. Combines attention-based dominant tok... | 212 |
| [vista-scene-aware-optimization-streaming](skills/vista-scene-aware-optimization-streaming/SKILL.md) | | | 261 |
| [vtc-r1-vision-text-compression-long-context](skills/vtc-r1-vision-text-compression-long-context/SKILL.md) | Implement VTC-R1 vision-text compression for efficient long-context reasoning. Renders intermediate reasoning segments i... | 269 |
| [weight-decay-improves-plasticity](skills/weight-decay-improves-plasticity/SKILL.md) | Configure weight decay for optimal model plasticity during LLM pretraining and fine-tuning. Advise on weight decay hyper... | 169 |
| [what-makes-low-bit-quantization-aware](skills/what-makes-low-bit-quantization-aware/SKILL.md) | Implement the Reasoning-QAT pipeline for low-bit quantization-aware training of reasoning LLMs. Combines PTQ initializat... | 189 |
| [when-get-significantly-worse](skills/when-get-significantly-worse/SKILL.md) | Statistically detect LLM degradation after optimization using McNemar's paired test. Use when: 'did quantization hurt my... | 245 |
| [who-deserves-reward-sharp](skills/who-deserves-reward-sharp/SKILL.md) | Apply SHARP (Shapley-based credit attribution) to design and optimize multi-agent systems where each agent's individual ... | 220 |
| [xlist-hate-checklist-based-framework-interpretable](skills/xlist-hate-checklist-based-framework-interpretable/SKILL.md) | Decompose hate speech detection into a checklist of ten concept-level binary questions answered independently by an LLM,... | 229 |
| [your-secretly-contains-personality](skills/your-secretly-contains-personality/SKILL.md) | Extract and activate persona-specialized subnetworks from LLMs using activation-guided pruning and contrastive masking. ... | 233 |
| [zero-sum-svd-balancing](skills/zero-sum-svd-balancing/SKILL.md) | Compress LLMs using Zero Sum SVD (ZS-SVD) — a post-training low-rank compression method that globally allocates heteroge... | 236 |
| [zipmoe-on-device-moe-serving](skills/zipmoe-on-device-moe-serving/SKILL.md) | Deploy Mixture-of-Experts LLMs on edge devices using lossless BF16 compression with bit-field decomposition and cache-af... | 213 |

---

## Reasoning & Chain-of-Thought

**259 skills**

| Skill | Description | Lines |
|-------|-------------|-------|
| [3d-space-as-scratchpad-editable](skills/3d-space-as-scratchpad-editable/SKILL.md) | Build agentic pipelines that use 3D scene layout as an intermediate reasoning workspace for controllable, spatially-accu... | 172 |
| [adareasoner-dynamic-tool-orchestration](skills/adareasoner-dynamic-tool-orchestration/SKILL.md) | Adaptive multi-step tool orchestration for complex reasoning tasks. Dynamically selects, sequences, and composes tools b... | 168 |
| [aero-autonomous-evolutionary-reasoning](skills/aero-autonomous-evolutionary-reasoning/SKILL.md) | Apply the AERO dual-loop self-evolution framework to iteratively improve reasoning on complex tasks. Uses entropy-based ... | 160 |
| [agentark-distilling-multi-agent-intelligence](skills/agentark-distilling-multi-agent-intelligence/SKILL.md) | Distill multi-agent debate reasoning into a single LLM's behavior. Apply AgentArk's three-tier distillation strategy to ... | 199 |
| [agentdrive-open-benchmark-dataset](skills/agentdrive-open-benchmark-dataset/SKILL.md) | Generate structured autonomous driving scenarios and MCQ benchmarks using AgentDrive's factorized 7-axis prompt-to-JSON ... | 284 |
| [agentic-reinforcement-learning-empowers](skills/agentic-reinforcement-learning-empowers/SKILL.md) | Build tool-augmented agent systems that decouple domain reasoning from knowledge storage, following the ChemCRAFT patter... | 242 |
| [agentic-very-long-video](skills/agentic-very-long-video/SKILL.md) | Build agentic systems for understanding very long video streams (hours to weeks) using entity scene graphs, multi-tool p... | 240 |
| [agenticsimlaw-juvenile-courtroom-multi-agent](skills/agenticsimlaw-juvenile-courtroom-multi-agent/SKILL.md) | Structured multi-agent courtroom debate for explainable high-stakes tabular decisions. Use when: 'set up a multi-agent d... | 181 |
| [agenttrace-structured-logging-framework](skills/agenttrace-structured-logging-framework/SKILL.md) | Implement structured, multi-surface observability logging for LLM agent systems using the AgentTrace pattern: operationa... | 152 |
| [an-cost-efficient-agentic-framework](skills/an-cost-efficient-agentic-framework/SKILL.md) | Audit Ethereum smart contracts for business logic vulnerabilities using Heimdallr's four-phase agentic pipeline: functio... | 202 |
| [audiorouter-data-audio-understanding](skills/audiorouter-data-audio-understanding/SKILL.md) | Build audio understanding systems that route between internal LLM reasoning and external audio tools using a lightweight... | 249 |
| [automating-computational-reproducibility-social](skills/automating-computational-reproducibility-social/SKILL.md) | Diagnose and repair failing computational research code to restore reproducibility. Uses an agent-based iterative workfl... | 189 |
| [autonomous-chain-of-thought-distillation-graph-bas](skills/autonomous-chain-of-thought-distillation-graph-bas/SKILL.md) | Implement FraudCoT-style graph-aware chain-of-thought distillation for fraud detection on text-attributed graphs. Combin... | 199 |
| [avere-improving-audiovisual-emotion](skills/avere-improving-audiovisual-emotion/SKILL.md) | Build emotion-aware multimodal AI systems that resist spurious cue-emotion associations and hallucinated audiovisual evi... | 177 |
| [bass-benchmarking-audio-lms](skills/bass-benchmarking-audio-lms/SKILL.md) | Build evaluation benchmarks for audio language models using the BASS methodology — structured task taxonomies across str... | 260 |
| [benchmarking-text-to-python-against-text-to-sql](skills/benchmarking-text-to-python-against-text-to-sql/SKILL.md) | Generate correct Python/Pandas code from natural language questions over tabular data, applying the Logic Completion Fra... | 188 |
| [beyond-alignment-expanding-reasoning](skills/beyond-alignment-expanding-reasoning/SKILL.md) | Apply Manifold-Reshaping Policy Optimization (MRPO) to expand LLM reasoning capacity beyond alignment. Implements spectr... | 250 |
| [beyond-confidence-rhythms-reasoning](skills/beyond-confidence-rhythms-reasoning/SKILL.md) | Analyze and improve LLM prompt robustness using the Token Constraint Bound (delta-TCB) metric from the paper 'Beyond Con... | 169 |
| [beyond-speedup-utilizing](skills/beyond-speedup-utilizing/SKILL.md) | Reuse LLM KV caches as free embeddings for confidence scoring and adaptive fast/slow reasoning. Use when: 'extract embed... | 247 |
| [biases-blind-spot-detecting](skills/biases-blind-spot-detecting/SKILL.md) | Automated black-box pipeline for detecting unverbalized biases in LLM decision-making. Discovers biases that models exhi... | 194 |
| [birdturk-adaptation-bird-text-to-sql](skills/birdturk-adaptation-bird-text-to-sql/SKILL.md) | Adapt Text-to-SQL systems and benchmarks for non-English, morphologically rich languages using controlled translation pi... | 242 |
| [bridging-arithmetic-gap-cognitive](skills/bridging-arithmetic-gap-cognitive/SKILL.md) | Iterative Dual-Phase Financial-PoT: decouple semantic reasoning from arithmetic computation to eliminate calculation err... | 222 |
| [bridging-lexical-ambiguity-vision](skills/bridging-lexical-ambiguity-vision/SKILL.md) | Build Visual Word Sense Disambiguation (VWSD) systems that resolve lexical ambiguity using CLIP, diffusion models, and L... | 188 |
| [bridging-modality-gap-roadside](skills/bridging-modality-gap-roadside/SKILL.md) | Build training-free pipelines that convert sparse 3D LiDAR point clouds into depth-encoded 2D images for classification ... | 211 |
| [can-implement-agent-based-odd-based](skills/can-implement-agent-based-odd-based/SKILL.md) | Translate ODD protocol specifications into validated, executable agent-based model (ABM) code in Python. Use when the us... | 208 |
| [can-post-training-transform-causal](skills/can-post-training-transform-causal/SKILL.md) | Perform rigorous causal inference tasks using structured reasoning pipelines inspired by CauGym. Estimate treatment effe... | 179 |
| [can-reasoning-be-trusted](skills/can-reasoning-be-trusted/SKILL.md) | Validate and score LLM-generated statistical reasoning using a three-axis rubric (Correctness 40%, Explanation 35%, Reas... | 203 |
| [can-we-classify-flaky](skills/can-we-classify-flaky/SKILL.md) | Analyze test suites for flaky tests using LLM-based classification with context-augmented reasoning. Applies findings fr... | 182 |
| [causalt5k-diagnosing-informing-refusal](skills/causalt5k-diagnosing-informing-refusal/SKILL.md) | Diagnose and correct causal reasoning failures in LLM outputs using the CausalT5K framework. Detects rung collapse (answ... | 161 |
| [chain-mindset-reasoning-adaptive](skills/chain-mindset-reasoning-adaptive/SKILL.md) | Solve complex problems by switching between four cognitive mindsets (Spatial, Convergent, Divergent, Algorithmic) at eac... | 177 |
| [chain-simulation-dual-mode-reasoning](skills/chain-simulation-dual-mode-reasoning/SKILL.md) | Dual-mode reasoning framework that dynamically routes problems to specialized strategies: computational flow for math, s... | 175 |
| [chatting-images-introspective-visual](skills/chatting-images-introspective-visual/SKILL.md) | Apply introspective visual thinking by iteratively 'chatting with images' — using language-guided re-examination of visu... | 186 |
| [closing-reasoning-gaps-clinical](skills/closing-reasoning-gaps-clinical/SKILL.md) | Build systems that detect and fix reasoning gaps in LLM agents by comparing their chain-of-thought against reference rea... | 196 |
| [co-redteam-orchestrated-security-discovery](skills/co-redteam-orchestrated-security-discovery/SKILL.md) | Multi-agent security vulnerability discovery and exploitation using Co-RedTeam's orchestrated workflow. Decomposes secur... | 197 |
| [codecircuit-inferring-llm-generated-code](skills/codecircuit-inferring-llm-generated-code/SKILL.md) | Assess LLM-generated code correctness using attribution graph analysis inspired by mechanistic interpretability. Apply s... | 195 |
| [cognitive-platform-engineering-autonomous](skills/cognitive-platform-engineering-autonomous/SKILL.md) | Build autonomous cloud operations using a four-plane cognitive architecture (Sensing, Reasoning, Orchestration, Experien... | 247 |
| [computational-approach-visual-metonymy](skills/computational-approach-visual-metonymy/SKILL.md) | Generate and evaluate visual metonymy -- indirect visual representations that evoke concepts through associated cues rat... | 179 |
| [controlling-output-rankings-generative](skills/controlling-output-rankings-generative/SKILL.md) | Optimize product/content descriptions to influence rankings in LLM-based search engines (generative engines) using the C... | 245 |
| [convexbench-recognize-convex-functions](skills/convexbench-recognize-convex-functions/SKILL.md) | Determine the convexity of arbitrarily deep symbolic function compositions using AST decomposition and recursive DCP-rul... | 161 |
| [cord-bridging-audio-text-reasoning](skills/cord-bridging-audio-text-reasoning/SKILL.md) | Implement CORD (Cross-modal On-policy Distillation) to bridge modality gaps in multimodal AI systems. Applies weighted s... | 238 |
| [core-comprehensive-ontological-relation](skills/core-comprehensive-ontological-relation/SKILL.md) | Detect and prevent semantic collapse in LLM outputs — where models fabricate spurious relationships between unrelated co... | 216 |
| [corefine-confidence-guided-self-refinement-adaptiv](skills/corefine-confidence-guided-self-refinement-adaptiv/SKILL.md) | Confidence-guided self-refinement for adaptive reasoning. Implements the CoRefine pattern: assess confidence in each rea... | 176 |
| [craft-calibrated-reasoning-answer-faithful](skills/craft-calibrated-reasoning-answer-faithful/SKILL.md) | Apply CRAFT (Calibrated Reasoning with Answer-Faithful Traces) for multi-hop question answering with verified reasoning ... | 192 |
| [ctrlcot-dual-granularity-chain-of-thought-compress](skills/ctrlcot-dual-granularity-chain-of-thought-compress/SKILL.md) | Compress chain-of-thought reasoning using CtrlCoT's dual-granularity framework: hierarchical semantic abstraction combin... | 157 |
| [cutting-gordian-knot-detecting](skills/cutting-gordian-knot-detecting/SKILL.md) | Detect malicious PyPI/NPM packages using behavioral pattern mining and semantic reasoning (PyGuard). Use when: 'scan thi... | 212 |
| [decomposing-reasoning-efficiency](skills/decomposing-reasoning-efficiency/SKILL.md) | > | 248 |
| [decoupled-reasoning-implicit-fact](skills/decoupled-reasoning-implicit-fact/SKILL.md) | Build dual-model pipelines that decouple knowledge extraction from reasoning over long documents. Compress document chun... | 164 |
| [decoupling-skeleton-flesh-multimodal](skills/decoupling-skeleton-flesh-multimodal/SKILL.md) | Disentangled structure-content reasoning for table images and structured data. Separates table skeleton (layout/structur... | 180 |
| [deep-search-hierarchical-meta-cognitive](skills/deep-search-hierarchical-meta-cognitive/SKILL.md) | Implement hierarchical meta-cognitive monitoring for deep search agents. Embeds a two-tier self-monitoring system (fast ... | 188 |
| [deepera-deep-evidence-reranking](skills/deepera-deep-evidence-reranking/SKILL.md) | Rerank retrieved passages for RAG pipelines using step-by-step logical reasoning to filter out semantically similar but ... | 223 |
| [deepimagesearch-benchmarking-multimodal-agents](skills/deepimagesearch-benchmarking-multimodal-agents/SKILL.md) | Build agentic image retrieval systems that perform multi-step contextual reasoning over visual histories instead of isol... | 198 |
| [deepplanning-benchmarking-long-horizon-agentic](skills/deepplanning-benchmarking-long-horizon-agentic/SKILL.md) | Solve long-horizon planning tasks with verifiable constraints using the DeepPlanning methodology: proactive information ... | 155 |
| [deepread-document-structure-aware-reasoning](skills/deepread-document-structure-aware-reasoning/SKILL.md) | | | 187 |
| [dep-search-learning-dependency-aware-reasoning](skills/dep-search-learning-dependency-aware-reasoning/SKILL.md) | Dependency-aware multi-step reasoning with persistent memory for complex questions requiring information retrieval acros... | 213 |
| [dispo-enhancing-training-efficiency](skills/dispo-enhancing-training-efficiency/SKILL.md) | Implement the DISPO reinforcement learning algorithm for training LLMs on mathematical reasoning with decoupled importan... | 277 |
| [distilling-reasoning-graph-concept](skills/distilling-reasoning-graph-concept/SKILL.md) | Distill LLM reasoning into a DAG of modular concept predictors for efficient, interpretable classification. Use when ask... | 170 |
| [dllm-searcher-adapting-diffusion-large](skills/dllm-searcher-adapting-diffusion-large/SKILL.md) | Implement the P-ReAct parallel reasoning-and-acting agent paradigm from DLLM-Searcher, which overlaps tool execution wit... | 256 |
| [do-reasoning-ask-questions](skills/do-reasoning-ask-questions/SKILL.md) | Information-theoretic question-asking framework for disambiguating user intent through structured yes/no questions. Uses... | 183 |
| [do-reasoning-enhance-embedding](skills/do-reasoning-enhance-embedding/SKILL.md) | | | 242 |
| [dr-mas-stable-reinforcement-learning](skills/dr-mas-stable-reinforcement-learning/SKILL.md) | Design and implement stable reinforcement learning pipelines for multi-agent LLM systems using agent-wise advantage norm... | 203 |
| [drugr-optimizing-molecular-drugs](skills/drugr-optimizing-molecular-drugs/SKILL.md) | Optimize molecular drug candidates using LLM-based explicit pharmacological reasoning over SMILES structures. Applies th... | 187 |
| [duogen-general-purpose-interleaved](skills/duogen-general-purpose-interleaved/SKILL.md) | Design and implement interleaved multimodal generation pipelines that alternate between text and image generation using ... | 209 |
| [dynamic-long-context-reasoning](skills/dynamic-long-context-reasoning/SKILL.md) | | | 210 |
| [ecco-evidence-driven-causal-reasoning](skills/ecco-evidence-driven-causal-reasoning/SKILL.md) | > | 205 |
| [ecg-r1-protocol-guided-modality-agnostic-mllm](skills/ecg-r1-protocol-guided-modality-agnostic-mllm/SKILL.md) | Build protocol-guided medical AI interpretation pipelines with structured diagnostic reasoning, modality-robust architec... | 260 |
| [eft-cot-multi-agent-chain-of-thought-framework](skills/eft-cot-multi-agent-chain-of-thought-framework/SKILL.md) | Build multi-agent emotion-focused therapy (EFT) reasoning pipelines for empathetic mental health Q&A systems. Uses a bot... | 300 |
| [eliciting-least-to-most-reasoning-phishing](skills/eliciting-least-to-most-reasoning-phishing/SKILL.md) | Detect phishing URLs using Least-to-Most iterative decomposition with answer sensitivity scoring. Triggers: 'analyze thi... | 155 |
| [emotion-llamav2-mmeverse-framework-benchmark](skills/emotion-llamav2-mmeverse-framework-benchmark/SKILL.md) | Build multimodal emotion understanding systems using the Emotion-LLaMAv2 architecture and MMEVerse benchmark methodology... | 232 |
| [emotionthinker-prosody-aware-reinforcement-learnin](skills/emotionthinker-prosody-aware-reinforcement-learnin/SKILL.md) | Build prosody-aware speech emotion reasoning pipelines using Chain-of-Thought RL. Implements EmotionThinker's GRPO-PTR t... | 292 |
| [empirical-mcts-continuous-agent-evolution](skills/empirical-mcts-continuous-agent-evolution/SKILL.md) | Applies Empirical-MCTS dual-loop reasoning: structured tree search with persistent memory that accumulates experience ac... | 195 |
| [enhancing-mathematical-problem-solving](skills/enhancing-mathematical-problem-solving/SKILL.md) | | | 183 |
| [entworld-holistic-environment-benchmark](skills/entworld-holistic-environment-benchmark/SKILL.md) | Build verifiable enterprise GUI agent benchmarks using schema-grounded task generation and SQL-based deterministic verif... | 158 |
| [es-memeval-benchmarking-conversational-agents](skills/es-memeval-benchmarking-conversational-agents/SKILL.md) | Build and evaluate long-term memory systems for conversational agents using the ES-MemEval five-capability framework (in... | 226 |
| [evaluating-enhancing-vulnerability-reasoning](skills/evaluating-enhancing-vulnerability-reasoning/SKILL.md) | Perform DAG-structured vulnerability reasoning on code, modeling causal dependencies between code facts instead of linea... | 211 |
| [evaluating-they-not-know](skills/evaluating-they-not-know/SKILL.md) | Build statistically efficient LLM evaluation pipelines that combine direct accuracy with pairwise comparison signals as ... | 185 |
| [evaluation-legal-applications-challenges](skills/evaluation-legal-applications-challenges/SKILL.md) | Build evaluation pipelines for LLMs in legal tasks using a three-dimensional framework: outcome correctness, reasoning r... | 171 |
| [evaluation-oncotimia-system-supporting](skills/evaluation-oncotimia-system-supporting/SKILL.md) | Build RAG pipelines that transform unstructured clinical or domain-specific documents into structured form records using... | 213 |
| [eventcast-hybrid-demand-forecasting](skills/eventcast-hybrid-demand-forecasting/SKILL.md) | Build hybrid demand forecasting systems that fuse LLM-extracted event knowledge with time-series models using a dual-tow... | 228 |
| [evolving-tool-user-creator](skills/evolving-tool-user-creator/SKILL.md) | Transform Claude from a static tool user into a dynamic tool creator using the UCT (User-to-Creator Transformation) fram... | 181 |
| [explainable-deepfake-detection-rl](skills/explainable-deepfake-detection-rl/SKILL.md) | Build explainable deepfake detection systems using RL-enhanced Self-Blended Images and Chain-of-Thought reasoning. Use w... | 296 |
| [exploring-reasoning-reward-agents](skills/exploring-reasoning-reward-agents/SKILL.md) | | | 219 |
| [fademem-biologically-inspired-forgetting-agent](skills/fademem-biologically-inspired-forgetting-agent/SKILL.md) | > | 222 |
| [fin-rate-real-world-financial-analytics](skills/fin-rate-real-world-financial-analytics/SKILL.md) | Analyze SEC filings and financial disclosures using the Fin-RATE three-pathway methodology: detail-oriented reasoning wi... | 182 |
| [flyaoc-evaluating-agentic-ontology](skills/flyaoc-evaluating-agentic-ontology/SKILL.md) | Build multi-agent systems for end-to-end ontology curation from scientific literature. Applies FlyAOC's agent architectu... | 184 |
| [forest-chat-adapting-vision-language-agents](skills/forest-chat-adapting-vision-language-agents/SKILL.md) | Build LLM-orchestrated agents for bi-temporal satellite image change analysis, combining vision-language models with too... | 362 |
| [from-assumptions-actions-turning](skills/from-assumptions-actions-turning/SKILL.md) | Build uncertainty-aware planners for multi-agent systems using the PCE (Planner-Composer-Evaluator) decision tree framew... | 242 |
| [from-consistency-complementarity-aligned](skills/from-consistency-complementarity-aligned/SKILL.md) | Build multi-modal time series analysis pipelines that align numerical data with visual plots and textual captions using ... | 230 |
| [from-gameplay-traces-game](skills/from-gameplay-traces-game/SKILL.md) | Reverse-engineer game mechanics from gameplay traces using a two-stage causal induction pipeline: first infer a Structur... | 211 |
| [from-passive-metric-active](skills/from-passive-metric-active/SKILL.md) | Build systems that use LLM uncertainty as an active control signal -- routing computation, triggering tool calls, enabli... | 268 |
| [from-perception-action-spatial](skills/from-perception-action-spatial/SKILL.md) | Design and implement spatially-aware AI agent systems using hierarchical memory, GNN-LLM integration, and world models. ... | 217 |
| [from-sparse-decisions-dense](skills/from-sparse-decisions-dense/SKILL.md) | Build content moderation and safety classification systems using multi-attribute trajectory reasoning instead of binary ... | 261 |
| [from-task-solving-robust](skills/from-task-solving-robust/SKILL.md) | Build LLM agent workflows that stay robust under partial observability, noisy signals, shifting environments, and intern... | 199 |
| [frost-filtering-reasoning-outliers](skills/frost-filtering-reasoning-outliers/SKILL.md) | Implement FROST (Filtering Reasoning Outliers with Attention) to prune unnecessary reasoning steps from LLM chain-of-tho... | 252 |
| [funprm-function-as-step-process-reward](skills/funprm-function-as-step-process-reward/SKILL.md) | Generate high-quality code by decomposing solutions into modular functions (Chain-of-Function style), then self-evaluati... | 251 |
| [generating-data-driven-reasoning-rubrics](skills/generating-data-driven-reasoning-rubrics/SKILL.md) | Build granular error taxonomies from incorrect reasoning traces, then use those rubrics to detect errors in LLM outputs ... | 170 |
| [genius-generative-fluid-intelligence](skills/genius-generative-fluid-intelligence/SKILL.md) | Evaluate and improve generative AI outputs for fluid intelligence tasks -- pattern induction from context, ad-hoc constr... | 247 |
| [graphagents-knowledge-graph-guided-agentic](skills/graphagents-knowledge-graph-guided-agentic/SKILL.md) | Build multi-agent pipelines that use knowledge graphs to guide LLM reasoning across domains. Agents specialize in proble... | 185 |
| [graphdancer-training-explore-reason](skills/graphdancer-training-explore-reason/SKILL.md) | Build agentic graph-exploration systems where an LLM navigates heterogeneous knowledge graphs through interleaved reason... | 243 |
| [graphseek-next-generation-graph-analytics](skills/graphseek-next-generation-graph-analytics/SKILL.md) | Build LLM-powered graph analytics systems using the GraphSeek two-plane architecture: a Semantic Catalog for planning ov... | 151 |
| [group-distributionally-robust-optimization-driven](skills/group-distributionally-robust-optimization-driven/SKILL.md) | Apply Group Distributionally Robust Optimization (GDRO) to RL-based LLM training. Dynamically classify prompts by diffic... | 205 |
| [he-snr-uncovering-latent-logic](skills/he-snr-uncovering-latent-logic/SKILL.md) | Evaluate and optimize LLM training data quality for software engineering tasks using the HE-SNR (High-Entropy Signal-to-... | 234 |
| [history-guided-iterative-visual-reasoning](skills/history-guided-iterative-visual-reasoning/SKILL.md) | | | 170 |
| [how-much-reasoning-retrieval-augmented](skills/how-much-reasoning-retrieval-augmented/SKILL.md) | Build contamination-aware hybrid RAG evaluation pipelines that couple knowledge graphs with text retrieval for multi-hop... | 178 |
| [how-personalized-memory-shape](skills/how-personalized-memory-shape/SKILL.md) | Rational preference utilization for personalized LLM assistants. Implements RP-Reasoner's pragmatic reasoning to selecti... | 216 |
| [hugrag-hierarchical-causal-knowledge](skills/hugrag-hierarchical-causal-knowledge/SKILL.md) | Build hierarchical causal knowledge graphs for RAG pipelines that suppress spurious correlations and enable cross-docume... | 168 |
| [identifying-adversary-tactics-techniques](skills/identifying-adversary-tactics-techniques/SKILL.md) | Identify MITRE ATT&CK Tactics, Techniques, and Procedures (TTPs) in decompiled malware binaries using the TTPDetect meth... | 198 |
| [iesr-mcts-based-modular-reasoning](skills/iesr-mcts-based-modular-reasoning/SKILL.md) | Convert natural language questions into SQL queries using MCTS-based modular reasoning inspired by the IESR framework. D... | 242 |
| [innovator-vl-multimodal-scientific-discovery](skills/innovator-vl-multimodal-scientific-discovery/SKILL.md) | Build data-efficient multimodal scientific ML pipelines using Innovator-VL's principled training methodology. Applies tr... | 247 |
| [internalizing-multi-agent-reasoning-accurate](skills/internalizing-multi-agent-reasoning-accurate/SKILL.md) | Distill multi-agent reasoning into a single efficient model for recommendation or retrieval. Use when: 'build a recommen... | 174 |
| [internalizing-reasoning-discovery-replay](skills/internalizing-reasoning-discovery-replay/SKILL.md) | Apply the STIR (Self-Distilled Tools for Internal Reasoning) pattern to build systems that discover reusable reasoning p... | 253 |
| [interpreting-controlling-reasoning-integrated](skills/interpreting-controlling-reasoning-integrated/SKILL.md) | Interpret and control LLM reasoning behavior using Integrated Policy Gradient (IPG) attribution. Identifies which intern... | 209 |
| [jailbreaks-vision-multimodal-reasoning](skills/jailbreaks-vision-multimodal-reasoning/SKILL.md) | > | 250 |
| [jobresqa-benchmark-machine-reading](skills/jobresqa-benchmark-machine-reading/SKILL.md) | Build and evaluate multilingual machine reading comprehension systems for HR documents (resumes and job descriptions). I... | 152 |
| [kid-knowledge-injected-dual-head-learning](skills/kid-knowledge-injected-dual-head-learning/SKILL.md) | Build knowledge-grounded multimodal content classifiers using the KID dual-head architecture: entity-anchored knowledge ... | 185 |
| [knowledge-graphs-implicit-reward](skills/knowledge-graphs-implicit-reward/SKILL.md) | Build compositional reasoning systems that use knowledge graph paths as reward signals to ground LLM reasoning in verifi... | 168 |
| [koral-knowledge-graph-guided](skills/koral-knowledge-graph-guided/SKILL.md) | Build Knowledge Graph-guided LLM reasoning pipelines for operational telemetry analysis. Combines a Literature KG (extra... | 247 |
| [krone-hierarchical-modular-log](skills/krone-hierarchical-modular-log/SKILL.md) | Detect anomalies in application logs using KRONE's hierarchical decomposition: parse flat log sequences into Entity/Acti... | 240 |
| [large-reasoning-failures](skills/large-reasoning-failures/SKILL.md) | Detect and mitigate known LLM reasoning failures during code generation, review, and problem-solving. Applies the taxono... | 218 |
| [large-scale-multidimensional-knowledge-profiling](skills/large-scale-multidimensional-knowledge-profiling/SKILL.md) | Build multidimensional profiling pipelines for large scientific paper corpora. Combines BERTopic clustering, LLM-structu... | 246 |
| [latent-chain-of-thought-as-planning](skills/latent-chain-of-thought-as-planning/SKILL.md) | Decouple reasoning from verbalization using PLaT-inspired latent planning. Maintains a broad solution space through para... | 200 |
| [latentchem-textual-cot-latent](skills/latentchem-textual-cot-latent/SKILL.md) | Apply LatentChem's latent-space reasoning paradigm to chemical computation tasks -- replacing verbose textual Chain-of-T... | 189 |
| [learning-compose-cross-domain-agentic](skills/learning-compose-cross-domain-agentic/SKILL.md) | Generate cross-domain agentic workflows using decompose-recompose-decide composition over reusable capability bases. Use... | 159 |
| [learning-decode-against-compositional](skills/learning-decode-against-compositional/SKILL.md) | Detect and mitigate compositional hallucinations in video multimodal LLM outputs using triple-pathway contrastive decodi... | 284 |
| [learning-irrecoverable-error-localized-policy](skills/learning-irrecoverable-error-localized-policy/SKILL.md) | Debug multi-step tool-using agent pipelines by localizing the first irrecoverable error via binary-search rollback, then... | 175 |
| [learning-reason-faithfully-step-level](skills/learning-reason-faithfully-step-level/SKILL.md) | Apply FaithRL's step-level faithfulness verification to multi-step reasoning tasks. Decomposes reasoning into individual... | 173 |
| [lec-kg-llm-embedding-collaborative-framework](skills/lec-kg-llm-embedding-collaborative-framework/SKILL.md) | Build domain-specific knowledge graphs from unstructured text using an iterative LLM + embedding validation loop. Combin... | 186 |
| [legalone-family-foundation-reliable](skills/legalone-family-foundation-reliable/SKILL.md) | Build domain-specialized LLM training pipelines using the LegalOne three-phase methodology: Plasticity-Adjusted Sampling... | 259 |
| [less-noise-more-voice](skills/less-noise-more-voice/SKILL.md) | Identify and remove interference tokens from prompts to improve LLM reasoning accuracy. Based on the LENS framework (Les... | 236 |
| [leveraging-turkish-skill-extraction](skills/leveraging-turkish-skill-extraction/SKILL.md) | Extract and normalize skills from job postings using a two-stage LLM pipeline: dynamic few-shot skill identification fol... | 198 |
| [lingxidiagbench-multi-agent-framework-benchmarking](skills/lingxidiagbench-multi-agent-framework-benchmarking/SKILL.md) | Build multi-agent benchmarking systems with role-separated agents (simulator, interviewer, evaluator) for structured mul... | 216 |
| [llama-31-foundationai-securityllm-reasoning-8b-tec](skills/llama-31-foundationai-securityllm-reasoning-8b-tec/SKILL.md) | > | 251 |
| [llm-assisted-logic-rule-learning](skills/llm-assisted-logic-rule-learning/SKILL.md) | Build deterministic, interpretable anomaly detection rule sets for time series data using LLM-driven labeling, symbolic ... | 181 |
| [llm-enhanced-reinforcement-learning-long-term](skills/llm-enhanced-reinforcement-learning-long-term/SKILL.md) | Build hierarchical recommendation systems that combine LLM semantic planning with RL fine-grained optimization for long-... | 247 |
| [llm-fsm-scaling-finite-state-reasoning](skills/llm-fsm-scaling-finite-state-reasoning/SKILL.md) | Generate correct RTL (Verilog/SystemVerilog) implementations of finite-state machines from natural-language specificatio... | 265 |
| [llms-as-high-dimensional-nonlinear](skills/llms-as-high-dimensional-nonlinear/SKILL.md) | Analyze, debug, and design LLM systems using the mathematical framework of high-dimensional nonlinear autoregressive mod... | 190 |
| [lmmrec-llm-driven-motivation-aware-multimodal](skills/lmmrec-llm-driven-motivation-aware-multimodal/SKILL.md) | Build motivation-aware recommendation systems that use LLM chain-of-thought prompting to extract user and item motivatio... | 217 |
| [logicscore-fine-grained-logic-evaluation](skills/logicscore-fine-grained-logic-evaluation/SKILL.md) | Evaluate the logical integrity of LLM-generated multi-hop answers using Horn Rule backward chaining. Scores Completeness... | 185 |
| [longcat-flash-thinking-2601-technical-report](skills/longcat-flash-thinking-2601-technical-report/SKILL.md) | Build robust multi-tool agentic pipelines with noise-aware execution, parallel reasoning, and environment scaling patter... | 311 |
| [magellan-autonomous-discovery-compiler](skills/magellan-autonomous-discovery-compiler/SKILL.md) | Evolve compiler optimization heuristics by coupling LLM code generation with evolutionary search and autotuning. Synthes... | 158 |
| [marble-multi-agent-reasoning-bioinformatics](skills/marble-multi-agent-reasoning-bioinformatics/SKILL.md) | Iteratively refine bioinformatics and ML models using MARBLE's multi-agent debate framework with role-specialized agents... | 206 |
| [martingale-foresight-sampling-principled](skills/martingale-foresight-sampling-principled/SKILL.md) | Implement Martingale Foresight Sampling (MFS) for principled LLM decoding with lookahead search. Replaces heuristic beam... | 220 |
| [mas-prove-understanding-process-verification](skills/mas-prove-understanding-process-verification/SKILL.md) | Design and implement process verification for multi-agent LLM systems. Add intermediate-step evaluation to multi-agent w... | 237 |
| [mascot-multi-agent-socio-collaborative-companion](skills/mascot-multi-agent-socio-collaborative-companion/SKILL.md) | Design and orchestrate multi-agent companion systems where each agent maintains a distinct persona and contributes diver... | 244 |
| [mata-multiagent-framework-for](skills/mata-multiagent-framework-for/SKILL.md) | Multi-agent table question answering using MATA's three-path reasoning strategy (Chain-of-Thought, Program-of-Thought, T... | 167 |
| [mata-trainable-hierarchical-automaton](skills/mata-trainable-hierarchical-automaton/SKILL.md) | Build multi-agent visual reasoning systems using hierarchical finite-state automata with a trainable hyper agent that or... | 303 |
| [mathliblemma-folklore-lemma-generation](skills/mathliblemma-folklore-lemma-generation/SKILL.md) | Multi-agent system for discovering and formalizing missing 'folklore' lemmas in Lean 4 / Mathlib. Identifies gaps in for... | 181 |
| [medmo-grounding-understanding-multimodal](skills/medmo-grounding-understanding-multimodal/SKILL.md) | Build medical image analysis pipelines with multi-stage grounded reasoning: cross-modal alignment, instruction-tuned VQA... | 313 |
| [medspeak-knowledge-graph-aided-asr](skills/medspeak-knowledge-graph-aided-asr/SKILL.md) | Build knowledge-graph-aided ASR error correction pipelines for medical speech, using phonetic similarity + semantic retr... | 262 |
| [medverse-reliable-medical-reasoning](skills/medverse-reliable-medical-reasoning/SKILL.md) | Decompose complex medical reasoning into DAG-structured parallel execution paths using Petri net theory. Improves accura... | 216 |
| [memcast-memory-driven-time-series](skills/memcast-memory-driven-time-series/SKILL.md) | Build memory-augmented time series forecasting systems using hierarchical experience storage (historical patterns, reaso... | 196 |
| [mermaid-memory-enhanced-retrieval-reasoning](skills/mermaid-memory-enhanced-retrieval-reasoning/SKILL.md) | Memory-enhanced multi-agent retrieval and reasoning for veracity assessment and fact-checking. Use when: 'verify this cl... | 189 |
| [metaphorstar-image-metaphor-understanding](skills/metaphorstar-image-metaphor-understanding/SKILL.md) | Analyze and interpret metaphorical, symbolic, and implied meaning in images using the MetaphorStar visual reasoning chai... | 197 |
| [mind-ambiguity-aleatoric-uncertainty](skills/mind-ambiguity-aleatoric-uncertainty/SKILL.md) | Detect ambiguous user queries in safety-critical QA systems using aleatoric uncertainty probes on LLM hidden states, the... | 225 |
| [mirror-multi-agent-framework-iterative](skills/mirror-multi-agent-framework-iterative/SKILL.md) | Translate natural language optimization problems into mathematical models and solver code using MIRROR's multi-agent pip... | 166 |
| [mixing-expert-knowledge-bring](skills/mixing-expert-knowledge-bring/SKILL.md) | Integrate domain expert knowledge into LLM fine-tuning pipelines using mixed cold-start SFT and reinforcement learning. ... | 199 |
| [mmts-bench-comprehensive-benchmark-time](skills/mmts-bench-comprehensive-benchmark-time/SKILL.md) | Evaluate and improve LLM performance on time series question-answering using the MMTS-BENCH hierarchical taxonomy. Cover... | 193 |
| [multi-agent-causal-reasoning-system](skills/multi-agent-causal-reasoning-system/SKILL.md) | Build multi-agent systems that discover causal rules from event sequences using specialized agents (causal discovery, co... | 225 |
| [multi-task-grpo-reliable-reasoning](skills/multi-task-grpo-reliable-reasoning/SKILL.md) | | | 228 |
| [multivis-agent-multi-agent-framework-logic](skills/multivis-agent-multi-agent-framework-logic/SKILL.md) | Build reliable multi-agent data visualization pipelines with logic rule constraints. Use when: 'generate a chart from th... | 193 |
| [nag-unified-native-architecture](skills/nag-unified-native-architecture/SKILL.md) | Encode graph structure directly into LM attention masks and positional IDs instead of using external GNN encoders. Use w... | 204 |
| [neural-theorem-proving-verification](skills/neural-theorem-proving-verification/SKILL.md) | Generate formal proofs for program verification conditions (VCs) in Isabelle, Lean 4, and Rocq. Translates C/WhyML code ... | 202 |
| [no-global-plan-chain-of-thought](skills/no-global-plan-chain-of-thought/SKILL.md) | Optimize Chain-of-Thought reasoning by detecting when CoT can be bypassed and identifying pivot positions that capture p... | 196 |
| [note2chat-improving-multi-turn-clinical](skills/note2chat-improving-multi-turn-clinical/SKILL.md) | Build structured multi-turn clinical history-taking agents and diagnostic chatbots using the Note2Chat framework: conver... | 175 |
| [omnirag-agent-agentic-omnimodal-reasoning](skills/omnirag-agent-agentic-omnimodal-reasoning/SKILL.md) | Build agentic multimodal RAG pipelines that answer questions over long audio-video content under resource constraints. U... | 278 |
| [optimal-turkish-subword-strategies](skills/optimal-turkish-subword-strategies/SKILL.md) | Design and evaluate subword tokenizers for Turkish and other morphologically rich languages (MRLs) using the vocabulary-... | 253 |
| [papersearchqa-learning-search-reason](skills/papersearchqa-learning-search-reason/SKILL.md) | Build iterative search-and-reason agents for scientific literature QA. Uses the PaperSearchQA pattern: interleaved think... | 233 |
| [parse-open-domain-reasoning-question](skills/parse-open-domain-reasoning-question/SKILL.md) | Build and evaluate reasoning-focused QA systems for low-resource languages using the PARSE methodology: structured promp... | 220 |
| [pathreasoner-r1-instilling-structured-reasoning](skills/pathreasoner-r1-instilling-structured-reasoning/SKILL.md) | Build knowledge-graph-guided structured reasoning pipelines for vision-language models in computational pathology. Imple... | 296 |
| [pcl-reasoner-v15-advancing-math-reasoning-offline](skills/pcl-reasoner-v15-advancing-math-reasoning-offline/SKILL.md) | Implement offline reinforcement learning pipelines for LLM reasoning tasks — decoupling data collection from training fo... | 255 |
| [persona-driven-data-synthesis-robust-multimodal](skills/persona-driven-data-synthesis-robust-multimodal/SKILL.md) | Generate synthetic training data using controllable persona-driven simulation and Chain-of-Thought reasoning augmentatio... | 161 |
| [phostream-benchmarking-real-world-streaming](skills/phostream-benchmarking-real-world-streaming/SKILL.md) | Build streaming multimodal benchmarks and evaluate omnimodal assistants on continuous audio-visual input with temporal r... | 238 |
| [physprover-advancing-automatic-theorem](skills/physprover-advancing-automatic-theorem/SKILL.md) | Build formal theorem proving pipelines for physics and scientific domains using conjecture-based data generation, Lean 4... | 218 |
| [polarmem-training-free-polarized-latent](skills/polarmem-training-free-polarized-latent/SKILL.md) | Build polarized memory systems for multimodal agents that encode both positive and negative evidence as graph constraint... | 183 |
| [pope-learning-reason-hard](skills/pope-learning-reason-hard/SKILL.md) | Apply the POPE (Privileged On-Policy Exploration) technique to solve hard reasoning problems by decomposing them with or... | 132 |
| [privacy-collapse-benign-fine-tuning](skills/privacy-collapse-benign-fine-tuning/SKILL.md) | Audit fine-tuning datasets and pipelines for privacy collapse — the silent failure where benign training data degrades a... | 195 |
| [prograph-r1-progress-aware-reinforcement-learning](skills/prograph-r1-progress-aware-reinforcement-learning/SKILL.md) | Build progress-aware GraphRAG agents that traverse knowledge graphs with structure-aware hypergraph retrieval and dense ... | 167 |
| [pruning-minimal-reasoning-graphs](skills/pruning-minimal-reasoning-graphs/SKILL.md) | > | 154 |
| [qrs-rule-synthesizing-neuro-symbolic-triad](skills/qrs-rule-synthesizing-neuro-symbolic-triad/SKILL.md) | Autonomous vulnerability discovery using the QRS (Query, Review, Sanitize) neuro-symbolic triad. Generates CodeQL querie... | 229 |
| [r1-syntheticvl-synthetic-data-generative](skills/r1-syntheticvl-synthetic-data-generative/SKILL.md) | Synthesize high-quality multimodal training data using Collective Adversarial Data Synthesis (CADS). Implements a cyclic... | 226 |
| [ragturk-best-practices-retrieval](skills/ragturk-best-practices-retrieval/SKILL.md) | Design and optimize RAG pipelines for Turkish and other morphologically rich languages (Turkish, Finnish, Hungarian, Kor... | 208 |
| [rank-and-reason-multi-agent-collaboration-accelera](skills/rank-and-reason-multi-agent-collaboration-accelera/SKILL.md) | | | 245 |
| [rapo-risk-aware-preference-optimization](skills/rapo-risk-aware-preference-optimization/SKILL.md) | Apply risk-aware preference optimization to make LLM reasoning chains safer against jailbreak attacks. Implements adapti... | 203 |
| [reasoning-augmented-representations-multimodal-ret](skills/reasoning-augmented-representations-multimodal-ret/SKILL.md) | Decouple reasoning from embedding compression in multimodal retrieval pipelines by enriching queries and corpus entries ... | 223 |
| [reasoning-beyond-literal-cross-style](skills/reasoning-beyond-literal-cross-style/SKILL.md) | Detect and interpret figurative language (sarcasm, humor, offense, metaphor) in multimodal image-text content using a st... | 176 |
| [reasoning-tool-use-compete-agentic](skills/reasoning-tool-use-compete-agentic/SKILL.md) | Diagnose and fix interference between reasoning and tool-use in agentic AI systems using LEAS attribution and DART-style... | 204 |
| [reasoning-while-asking-transforming](skills/reasoning-while-asking-transforming/SKILL.md) | | | 190 |
| [reducing-costs-proof-synthesis](skills/reducing-costs-proof-synthesis/SKILL.md) | Generate formally verified Rust code with Verus specifications and proofs using the VeruSyn methodology. Applies self-sy... | 247 |
| [redvisor-reasoning-aware-prompt-injection](skills/redvisor-reasoning-aware-prompt-injection/SKILL.md) | Defend LLM applications against prompt injection using RedVisor's two-phase reasoning-then-responding architecture. Impl... | 223 |
| [refer-agent-collaborative-multi-agent-system](skills/refer-agent-collaborative-multi-agent-system/SKILL.md) | Build collaborative multi-agent systems that use alternating reasoning-reflection cycles with specialized agent roles, c... | 179 |
| [reflect-transparent-principle-guided-reasoning](skills/reflect-transparent-principle-guided-reasoning/SKILL.md) | > | 185 |
| [regular-variational-latent-reasoning](skills/regular-variational-latent-reasoning/SKILL.md) | Compress verbose chain-of-thought reasoning into compact latent state representations guided by rendered visual summarie... | 236 |
| [report-nsf-workshop-ai](skills/report-nsf-workshop-ai/SKILL.md) | Apply AI techniques from the NSF AI-for-EDA workshop to hardware design tasks: RTL code generation from natural language... | 276 |
| [resagent-entropy-based-prior-point](skills/resagent-entropy-based-prior-point/SKILL.md) | Implement entropy-guided coarse-to-fine visual grounding pipelines for referring expression segmentation and point-promp... | 266 |
| [rethinker-scientific-reasoning-rethinking](skills/rethinker-scientific-reasoning-rethinking/SKILL.md) | Solve hard scientific and technical reasoning problems using the ReThinker Solver-Critic-Selector loop with confidence-g... | 239 |
| [rethinking-llm-as-a-judge-representation-as-a-judg](skills/rethinking-llm-as-a-judge-representation-as-a-judg/SKILL.md) | Build probing-based evaluation pipelines that judge LLM output quality using hidden-state representations from small lan... | 160 |
| [reward-designs-general-reasoning](skills/reward-designs-general-reasoning/SKILL.md) | Design and implement likelihood-based reward functions for RL fine-tuning of LLMs on reasoning tasks. Use when: 'design ... | 228 |
| [rewards-as-labels-revisiting](skills/rewards-as-labels-revisiting/SKILL.md) | Implement the REAL (Rewards as Labels) framework for LLM reinforcement learning, which reformulates RLVR policy optimiza... | 218 |
| [rpo-rag-aligning-small-relation-aware](skills/rpo-rag-aligning-small-relation-aware/SKILL.md) | Build knowledge-graph-grounded RAG pipelines that align small LLMs (under 8B params) with relation-aware preference opti... | 259 |
| [rubberduckbench-benchmark-ai-coding](skills/rubberduckbench-benchmark-ai-coding/SKILL.md) | Evaluate and improve AI coding assistant responses using RubberDuckBench's rubric-based methodology. Detects hallucinati... | 182 |
| [s3-cot-self-sampled-succinct-reasoning](skills/s3-cot-self-sampled-succinct-reasoning/SKILL.md) | Apply dual-cognitive reasoning (System 1 fast / System 2 slow) to compress verbose chain-of-thought into succinct, effic... | 201 |
| [scaling-medical-reasoning-verification](skills/scaling-medical-reasoning-verification/SKILL.md) | > | 191 |
| [scratcheval-multimodal-evaluation-framework](skills/scratcheval-multimodal-evaluation-framework/SKILL.md) | Evaluate, debug, and repair block-based Scratch programs using a three-layer executable protocol (VM execution, block-le... | 156 |
| [self-hinting-enhance-reinforcement-learning](skills/self-hinting-enhance-reinforcement-learning/SKILL.md) | Apply the SAGE self-hinting technique to improve LLM problem-solving by generating graduated hints that boost solution d... | 174 |
| [semanticalli-caching-reasoning-not](skills/semanticalli-caching-reasoning-not/SKILL.md) | Implement pipeline-aware intermediate representation (IR) caching for agentic systems. Instead of caching final LLM resp... | 202 |
| [sifting-noise-comparative-study](skills/sifting-noise-comparative-study/SKILL.md) | Filter false positives from static analysis security tools (SAST) using LLM-agent-driven triage. Applies iterative code ... | 154 |
| [socratic-geo-synthetic-data-generation](skills/socratic-geo-synthetic-data-generation/SKILL.md) | Generate high-quality synthetic training data through multi-agent feedback loops where a Teacher agent creates parameter... | 226 |
| [sogk-one-token-explicit-graph](skills/sogk-one-token-explicit-graph/SKILL.md) | Represent graph topology as a single discrete token (<SOG_k>) for LLM reasoning, replacing verbose graph verbalization. ... | 128 |
| [sonic-o1-real-world-benchmark-evaluating](skills/sonic-o1-real-world-benchmark-evaluating/SKILL.md) | Evaluate multimodal LLMs on audio-video understanding using the SONIC-O1 benchmark framework. Covers three task types: v... | 238 |
| [sparc-separating-perception-reasoning](skills/sparc-separating-perception-reasoning/SKILL.md) | | | 170 |
| [spd-faith-bench-diagnosing-improving](skills/spd-faith-bench-diagnosing-improving/SKILL.md) | Diagnose and improve faithfulness of chain-of-thought reasoning in multimodal LLM pipelines using the SPD-Faith Bench me... | 237 |
| [spectral-guardrails-agents-wild](skills/spectral-guardrails-agents-wild/SKILL.md) | Implement training-free hallucination detection for LLM agent tool calls using spectral analysis of attention topology. ... | 250 |
| [spotagent-grounding-visual-geo-localization](skills/spotagent-grounding-visual-geo-localization/SKILL.md) | Build agentic geo-localization systems that combine vision-language model reasoning with tool-assisted verification usin... | 248 |
| [sql-trail-multi-turn-reinforcement-learning](skills/sql-trail-multi-turn-reinforcement-learning/SKILL.md) | Iterative multi-turn Text-to-SQL generation using reason-execute-observe loops with execution feedback. Instead of writi... | 186 |
| [st-raptor-agentic-system-semi-structured](skills/st-raptor-agentic-system-semi-structured/SKILL.md) | Agentic system for answering questions about semi-structured tables using tree-based structural modeling and multi-step ... | 222 |
| [stalled-biased-confused-uncovering-reasoning](skills/stalled-biased-confused-uncovering-reasoning/SKILL.md) | Systematic root cause analysis for cloud/distributed system failures using a 16-category reasoning failure taxonomy and ... | 168 |
| [state-transition-framework-reasoning](skills/state-transition-framework-reasoning/SKILL.md) | | | 215 |
| [steer2adapt-dynamically-composing-steering](skills/steer2adapt-dynamically-composing-steering/SKILL.md) | Implement the Steer2Adapt framework for adapting LLMs at inference time by dynamically composing steering vectors from a... | 209 |
| [steuerllm-local-specialized-german](skills/steuerllm-local-specialized-german/SKILL.md) | Build domain-specialized LLM pipelines for formal-rule domains (tax law, legal, regulatory) using retrieval-augmented sy... | 203 |
| [strong-reasoning-isnt-enough](skills/strong-reasoning-isnt-enough/SKILL.md) | Build interactive diagnostic agents that systematically elicit evidence before concluding, using the REFINE (Reasoning-E... | 251 |
| [svrepair-structured-visual-reasoning](skills/svrepair-structured-visual-reasoning/SKILL.md) | Fix bugs using structured visual reasoning -- converts screenshots, control-flow graphs, and UI artifacts into semantic ... | 193 |
| [synthagent-multi-agent-framework-realistic](skills/synthagent-multi-agent-framework-realistic/SKILL.md) | Build multi-agent pipelines that generate realistic synthetic patient profiles by integrating epidemiological data, medi... | 298 |
| [synthesizing-file-level-data-unit](skills/synthesizing-file-level-data-unit/SKILL.md) | Generate high-quality unit tests with self-debugging repair loops and chain-of-thought reasoning. Produces tests with me... | 222 |
| [tangrampuzzle-evaluating-multimodal-compositional](skills/tangrampuzzle-evaluating-multimodal-compositional/SKILL.md) | Evaluate and build compositional spatial reasoning systems using geometry-grounded benchmarks and symbolic coordinate fr... | 233 |
| [task-oriented-robot-human-handovers-legged](skills/task-oriented-robot-human-handovers-legged/SKILL.md) | Implement task-oriented robot-to-human object handover systems using LLM-driven affordance reasoning and exemplar-based ... | 261 |
| [teaching-evaluating-reason-about](skills/teaching-evaluating-reason-about/SKILL.md) | Apply knowledge-augmented reasoning distillation for polymer design tasks. Builds structured Chain-of-Thought pipelines ... | 202 |
| [temp-r1-unified-autonomous-agent](skills/temp-r1-unified-autonomous-agent/SKILL.md) | Build autonomous agents that answer complex temporal questions over knowledge graphs or time-stamped datasets using stru... | 200 |
| [text-summarization-global-structure](skills/text-summarization-global-structure/SKILL.md) | Summarize long documents while preserving global semantic structure and logical coherence using topology-guided pruning ... | 165 |
| [the-clef-2026-finmmeval-lab](skills/the-clef-2026-finmmeval-lab/SKILL.md) | Build multilingual, multimodal financial AI evaluation pipelines using the FinMMEval framework. Covers financial exam QA... | 246 |
| [thinking-broad-acting-fast](skills/thinking-broad-acting-fast/SKILL.md) | Build production search relevance systems using Multi-Perspective Chain-of-Thought distillation into lightweight student... | 210 |
| [thinking-frames-visual-context](skills/thinking-frames-visual-context/SKILL.md) | Decompose complex visual reasoning and spatial planning tasks into frame-by-frame intermediate steps, using visual conte... | 238 |
| [timeblind-spatio-temporal-compositionality-benchma](skills/timeblind-spatio-temporal-compositionality-benchma/SKILL.md) | Build and evaluate spatio-temporal reasoning benchmarks for video LLMs using the TimeBlind minimal-pairs methodology. Ge... | 233 |
| [timely-machine-awareness-time](skills/timely-machine-awareness-time/SKILL.md) | Apply time-budget-aware reasoning to agentic tasks with tool calls. Dynamically adjust strategy depth, tool call frequen... | 171 |
| [tkg-thinker-dynamic-reasoning-over](skills/tkg-thinker-dynamic-reasoning-over/SKILL.md) | > | 244 |
| [toward-cognitive-supersensing-multimodal](skills/toward-cognitive-supersensing-multimodal/SKILL.md) | Apply Cognitive Supersensing to multimodal reasoning tasks -- augmenting text-only chain-of-thought with latent visual r... | 173 |
| [toward-culturally-aligned-ontology-guided](skills/toward-culturally-aligned-ontology-guided/SKILL.md) | Ontology-guided multi-agent reasoning for culturally aligned LLM outputs. Use when building systems that must respect cu... | 190 |
| [towards-autonomous-mathematics-research](skills/towards-autonomous-mathematics-research/SKILL.md) | Iterative generate-verify-revise agent for mathematical research problems. Implements the Aletheia loop: decompose a har... | 239 |
| [trapped-past-disentangling-fluid](skills/trapped-past-disentangling-fluid/SKILL.md) | Diagnose whether an LLM is memorizing or reasoning by constructing distributional proximity tests. Classifies task input... | 174 |
| [ts-debate-multimodal-collaborative-debate](skills/ts-debate-multimodal-collaborative-debate/SKILL.md) | Zero-shot time series reasoning via modality-specialized multi-agent debate. Assigns dedicated text, visual, and numeric... | 232 |
| [tsrbench-comprehensive-multi-task-multi-modal](skills/tsrbench-comprehensive-multi-task-multi-modal/SKILL.md) | Evaluate and build multi-modal time series reasoning pipelines using the TSRBench framework. Covers perception, reasonin... | 206 |
| [ttcs-test-time-curriculum-synthesis](skills/ttcs-test-time-curriculum-synthesis/SKILL.md) | Implement a co-evolving test-time curriculum synthesis framework where a question synthesizer and reasoning solver itera... | 226 |
| [tutorial-reasoning-ir-ir](skills/tutorial-reasoning-ir-ir/SKILL.md) | Build reasoning-enhanced information retrieval pipelines that go beyond semantic matching. Applies five methodological f... | 249 |
| [unicog-uncovering-cognitive-abilities](skills/unicog-uncovering-cognitive-abilities/SKILL.md) | Analyze and diagnose LLM reasoning through latent cognitive ability decomposition inspired by the UniCog framework. Deco... | 178 |
| [unveiling-cognitive-compass-theory-of-mind-guided](skills/unveiling-cognitive-compass-theory-of-mind-guided/SKILL.md) | Apply Theory-of-Mind (ToM) guided reasoning chains to multimodal emotion analysis tasks. Decomposes emotional reasoning ... | 197 |
| [urdubench-urdu-reasoning-benchmark](skills/urdubench-urdu-reasoning-benchmark/SKILL.md) | Build high-quality reasoning benchmarks for Urdu and other low-resource languages using contextually ensembled translati... | 171 |
| [verge-formal-refinement-guidance](skills/verge-formal-refinement-guidance/SKILL.md) | Iterative verification-guided reasoning that decomposes answers into atomic claims, classifies and routes them to formal... | 185 |
| [veri-sure-contract-aware-multi-agent-framework](skills/veri-sure-contract-aware-multi-agent-framework/SKILL.md) | Generate functionally correct RTL/Verilog code using a contract-aware multi-agent workflow with formal verification. Tri... | 186 |
| [videothinker-building-agentic-videollms](skills/videothinker-building-agentic-videollms/SKILL.md) | Build agentic video understanding systems with LLM-guided tool reasoning. Implements the VideoThinker pattern: confidenc... | 204 |
| [vihermes-graph-grounded-multihop-question](skills/vihermes-graph-grounded-multihop-question/SKILL.md) | Build graph-grounded multihop QA systems over regulatory and hierarchically structured documents. Combines vector simila... | 264 |
| [villain-at-averimatec-verifying](skills/villain-at-averimatec-verifying/SKILL.md) | Build multi-agent fact-checking pipelines that verify image-text claims through modality-specific analysis, cross-modal ... | 248 |
| [vision-deepresearch-incentivizing-deepresearch-cap](skills/vision-deepresearch-incentivizing-deepresearch-cap/SKILL.md) | Multi-turn, multi-entity, multi-scale visual and textual deep research agent for answering complex questions about image... | 180 |
| [visor-visual-spatial-object](skills/visor-visual-spatial-object/SKILL.md) | Implement VISOR-style three-stage visual spatial reasoning (think, think-summary, action) for embodied navigation and ob... | 198 |
| [vistira-closing-image-text-modality](skills/vistira-closing-image-text-modality/SKILL.md) | Solve math problems from images by decomposing them into interleaved natural-language rationales and executable Python c... | 189 |
| [visual-reasoning-over-time](skills/visual-reasoning-over-time/SKILL.md) | Analyze time series data using the MAS4TS Analyzer-Reasoner-Executor multi-agent paradigm: convert series to plots, extr... | 180 |
| [vtc-r1-vision-text-compression-long-context](skills/vtc-r1-vision-text-compression-long-context/SKILL.md) | Implement VTC-R1 vision-text compression for efficient long-context reasoning. Renders intermediate reasoning segments i... | 269 |
| [vulread-knowledge-graph-guided-software-vulnerabil](skills/vulread-knowledge-graph-guided-software-vulnerabil/SKILL.md) | CWE-guided vulnerability reasoning and detection using knowledge-graph-structured analysis. Analyzes source code for sec... | 212 |
| [wdscaling-parallel-tool-calling-deep](skills/wdscaling-parallel-tool-calling-deep/SKILL.md) | Scale deep research tasks by issuing parallel tool calls (width) alongside sequential reasoning (depth), following the W... | 161 |
| [what-makes-low-bit-quantization-aware](skills/what-makes-low-bit-quantization-aware/SKILL.md) | Implement the Reasoning-QAT pipeline for low-bit quantization-aware training of reasoning LLMs. Combines PTQ initializat... | 189 |
| [when-iterative-rag-beats](skills/when-iterative-rag-beats/SKILL.md) | Build iterative retrieval-reasoning RAG pipelines that outperform single-shot retrieval, using staged evidence gathering... | 244 |
| [why-reasoning-fails-plan](skills/why-reasoning-fails-plan/SKILL.md) | Apply FLARE (Future-aware Lookahead with Reward Estimation) to long-horizon coding tasks. Replaces greedy step-by-step r... | 218 |

---

## RAG & Retrieval

**207 skills**

| Skill | Description | Lines |
|-------|-------------|-------|
| [a-mapreduce-executing-wide-search](skills/a-mapreduce-executing-wide-search/SKILL.md) | Execute large-scale breadth-oriented search and retrieval tasks using the A-MapReduce pattern: decompose a wide query in... | 196 |
| [a-rag-scaling-agentic-retrieval-augmented](skills/a-rag-scaling-agentic-retrieval-augmented/SKILL.md) | > | 253 |
| [a2rag-adaptive-agentic-graph](skills/a2rag-adaptive-agentic-graph/SKILL.md) | Build adaptive, cost-aware Graph-RAG pipelines that route queries through escalating retrieval stages (local -> bridge -... | 229 |
| [aacr-bench-evaluating-automatic-code](skills/aacr-bench-evaluating-automatic-code/SKILL.md) | Perform repository-level automated code review on pull requests using hierarchical context retrieval and structured defe... | 187 |
| [accelerating-social-science-research](skills/accelerating-social-science-research/SKILL.md) | Implement the EXPERIGEN agentic framework for automated hypothesis generation and empirical validation on datasets. Uses... | 177 |
| [affective-flow-emotional-support](skills/affective-flow-emotional-support/SKILL.md) | Build emotionally supportive multi-turn conversation systems using the AFlow framework — affective flow modeling with MC... | 216 |
| [agent-based-software-artifact-evaluation](skills/agent-based-software-artifact-evaluation/SKILL.md) | Automatically evaluate software research artifacts (code repositories with READMEs) by constructing dependency-aware com... | 203 |
| [agentic-reinforcement-learning-empowers](skills/agentic-reinforcement-learning-empowers/SKILL.md) | Build tool-augmented agent systems that decouple domain reasoning from knowledge storage, following the ChemCRAFT patter... | 242 |
| [agentic-very-long-video](skills/agentic-very-long-video/SKILL.md) | Build agentic systems for understanding very long video streams (hours to weeks) using entity scene graphs, multi-tool p... | 240 |
| [agenttrace-structured-logging-framework](skills/agenttrace-structured-logging-framework/SKILL.md) | Implement structured, multi-surface observability logging for LLM agent systems using the AgentTrace pattern: operationa... | 152 |
| [agentxray-white-boxing-agentic-systems](skills/agentxray-white-boxing-agentic-systems/SKILL.md) | Reverse-engineer black-box agentic systems into editable, interpretable workflows using search-based reconstruction. Use... | 168 |
| [agyn-multi-agent-system-team-based](skills/agyn-multi-agent-system-team-based/SKILL.md) | Orchestrate multi-agent teams for autonomous software engineering using the Agyn methodology: coordinator, researcher, i... | 202 |
| [aiano-enhancing-information-retrieval](skills/aiano-enhancing-information-retrieval/SKILL.md) | Build AI-augmented annotation pipelines for creating high-quality information retrieval and QA datasets. Combines LLM-ge... | 161 |
| [aligncoder-aligning-retrieval-target](skills/aligncoder-aligning-retrieval-target/SKILL.md) | | | 206 |
| [alphaface-high-fidelity-real-time](skills/alphaface-high-fidelity-real-time/SKILL.md) | Search and retrieve information about AlphaFace, a real-time face-swapping architecture that uses CLIP contrastive losse... | 225 |
| [ama-adaptive-memory-multi-agent](skills/ama-adaptive-memory-multi-agent/SKILL.md) | Build adaptive memory systems using coordinated multi-agent collaboration with hierarchical storage and consistency main... | 224 |
| [amem4rec-leveraging-cross-user-similarity](skills/amem4rec-leveraging-cross-user-similarity/SKILL.md) | Build agentic recommendation systems that learn collaborative filtering signals through cross-user memory evolution -- n... | 273 |
| [analyticsgpt-workflow-scientometric-question](skills/analyticsgpt-workflow-scientometric-question/SKILL.md) | Build sequential LLM pipelines for scientometric question answering over academic databases. Decomposes meta-scientific ... | 306 |
| [arkeval-benchmarking-evaluating-automated](skills/arkeval-benchmarking-evaluating-automated/SKILL.md) | Automated ArkTS code repair using retrieval-augmented generation, LLM-based test oracle synthesis, and structured benchm... | 196 |
| [assessing-business-process-modeling](skills/assessing-business-process-modeling/SKILL.md) | Evaluate and generate BPMN process models from natural language using the BEF4LLM framework. Assess BPMN XML quality acr... | 229 |
| [automated-customization-enterprise-code](skills/automated-customization-enterprise-code/SKILL.md) | Customize LLMs for enterprise code repositories using semantic scopes -- automatically partition codebases into meaningf... | 153 |
| [automated-rubrics-reliable-evaluation](skills/automated-rubrics-reliable-evaluation/SKILL.md) | Generate fine-grained evaluation rubrics for medical dialogue systems using a retrieval-augmented multi-agent pipeline. ... | 180 |
| [automating-computational-reproducibility-social](skills/automating-computational-reproducibility-social/SKILL.md) | Diagnose and repair failing computational research code to restore reproducibility. Uses an agent-based iterative workfl... | 189 |
| [bear-beam-search-aware-optimization-recommendation](skills/bear-beam-search-aware-optimization-recommendation/SKILL.md) | Implement BEAR (Beam-SEarch-Aware Regularization) to fix training-inference mismatch in LLM-based recommendation systems... | 220 |
| [beyond-blame-rethinking-szz](skills/beyond-blame-rethinking-szz/SKILL.md) | Identify bug-inducing commits using temporal knowledge graph search beyond git blame. Use when: 'find what commit introd... | 154 |
| [beyond-confidence-rhythms-reasoning](skills/beyond-confidence-rhythms-reasoning/SKILL.md) | Analyze and improve LLM prompt robustness using the Token Constraint Bound (delta-TCB) metric from the paper 'Beyond Con... | 169 |
| [beyond-needles-illusion-decoupled](skills/beyond-needles-illusion-decoupled/SKILL.md) | Decouple evidence access from evidence use when evaluating or building long-context and RAG systems under semantic inter... | 191 |
| [breaking-static-graph-context-aware](skills/breaking-static-graph-context-aware/SKILL.md) | Build query-adaptive knowledge graph retrieval systems using CatRAG's context-aware traversal. Transforms static KG-base... | 170 |
| [cgpt-cluster-guided-partial-tables](skills/cgpt-cluster-guided-partial-tables/SKILL.md) | Improve table retrieval by constructing cluster-guided partial tables and generating synthetic training queries with LLM... | 230 |
| [chunking-retrieval-re-ranking-empirical-evaluation](skills/chunking-retrieval-re-ranking-empirical-evaluation/SKILL.md) | Build and optimize two-stage RAG pipelines with bi-encoder retrieval, cross-encoder re-ranking, and empirically-validate... | 230 |
| [cimrag-cim-aware-domain-adaptive-noise-resilient](skills/cimrag-cim-aware-domain-adaptive-noise-resilient/SKILL.md) | Build noise-resilient RAG retrieval pipelines for edge/resource-constrained deployments. Implements TONEL (Task-Oriented... | 227 |
| [closing-reasoning-gaps-clinical](skills/closing-reasoning-gaps-clinical/SKILL.md) | Build systems that detect and fix reasoning gaps in LLM agents by comparing their chain-of-thought against reference rea... | 196 |
| [compact-hypercube-embeddings-fast](skills/compact-hypercube-embeddings-fast/SKILL.md) | Build fast similarity-search systems using compact binary hypercube embeddings derived from foundation model encoders. R... | 192 |
| [compactrag-reducing-calls-token](skills/compactrag-reducing-calls-token/SKILL.md) | | | 194 |
| [comprehensive-comparison-rag-methods](skills/comprehensive-comparison-rag-methods/SKILL.md) | Select and configure the right RAG strategy for conversational QA systems based on dataset characteristics. Use when: 'b... | 167 |
| [contextual-drag-errors-context](skills/contextual-drag-errors-context/SKILL.md) | > | 249 |
| [controlling-output-rankings-generative](skills/controlling-output-rankings-generative/SKILL.md) | Optimize product/content descriptions to influence rankings in LLM-based search engines (generative engines) using the C... | 245 |
| [cost-efficient-rag-entity-matching](skills/cost-efficient-rag-entity-matching/SKILL.md) | Build cost-efficient RAG pipelines for entity matching and deduplication using blocking-based batch retrieval and genera... | 187 |
| [covagent-overcoming-30-curse](skills/covagent-overcoming-30-curse/SKILL.md) | Boost Android app test coverage beyond the 30% activity ceiling using agentic static analysis of Smali code, component t... | 174 |
| [craft-calibrated-reasoning-answer-faithful](skills/craft-calibrated-reasoning-answer-faithful/SKILL.md) | Apply CRAFT (Calibrated Reasoning with Answer-Faithful Traces) for multi-hop question answering with verified reasoning ... | 192 |
| [cua-skill-develop-skills-computer](skills/cua-skill-develop-skills-computer/SKILL.md) | Build reusable, parameterized skill libraries for computer-using agents (CUAs). Decomposes GUI automation into Skill Cel... | 297 |
| [curiosity-driven-knowledge-retrieval](skills/curiosity-driven-knowledge-retrieval/SKILL.md) | | | 219 |
| [cvedrl-code-verifier-difficulty-aware](skills/cvedrl-code-verifier-difficulty-aware/SKILL.md) | Generate difficulty-aware unit tests that verify LLM-generated code using branch coverage analysis, complexity-weighted ... | 195 |
| [darl-encouraging-diverse-answers](skills/darl-encouraging-diverse-answers/SKILL.md) | Generate diverse, high-quality answer variants for open-ended tasks using DARL's bounded-diversity framework. Use when: ... | 286 |
| [darwin-dynamic-agentically-rewriting](skills/darwin-dynamic-agentically-rewriting/SKILL.md) | Evolutionary multi-agent code optimization using genetic algorithms. Agents mutate each other's training/configuration c... | 167 |
| [deep-researcher-sequential-plan](skills/deep-researcher-sequential-plan/SKILL.md) | | | 194 |
| [deep-search-hierarchical-meta-cognitive](skills/deep-search-hierarchical-meta-cognitive/SKILL.md) | Implement hierarchical meta-cognitive monitoring for deep search agents. Embeds a two-tier self-monitoring system (fast ... | 188 |
| [deepera-deep-evidence-reranking](skills/deepera-deep-evidence-reranking/SKILL.md) | Rerank retrieved passages for RAG pipelines using step-by-step logical reasoning to filter out semantically similar but ... | 223 |
| [deepimagesearch-benchmarking-multimodal-agents](skills/deepimagesearch-benchmarking-multimodal-agents/SKILL.md) | Build agentic image retrieval systems that perform multi-step contextual reasoning over visual histories instead of isol... | 198 |
| [deltaevolve-accelerating-scientific-discovery](skills/deltaevolve-accelerating-scientific-discovery/SKILL.md) | Iteratively evolve code solutions using momentum-driven semantic deltas instead of full-code histories. Use when: 'evolv... | 187 |
| [dep-search-learning-dependency-aware-reasoning](skills/dep-search-learning-dependency-aware-reasoning/SKILL.md) | Dependency-aware multi-step reasoning with persistent memory for complex questions requiring information retrieval acros... | 213 |
| [diffusion-lms-approximate-optimal](skills/diffusion-lms-approximate-optimal/SKILL.md) | Search and retrieve research on Calibrated Adaptive Length (CAL) for diffusion language model infilling. Helps users und... | 170 |
| [diffusion-pretrained-dense-contextual-embeddings](skills/diffusion-pretrained-dense-contextual-embeddings/SKILL.md) | Build production retrieval systems using pplx-embed, diffusion-pretrained dense and contextualized embedding models with... | 183 |
| [diverge-diversity-enhanced-rag-open-ended](skills/diverge-diversity-enhanced-rag-open-ended/SKILL.md) | Diversity-enhanced RAG for open-ended queries with multiple valid answers. Uses reflection-guided generation and memory-... | 177 |
| [dllm-searcher-adapting-diffusion-large](skills/dllm-searcher-adapting-diffusion-large/SKILL.md) | Implement the P-ReAct parallel reasoning-and-acting agent paradigm from DLLM-Searcher, which overlaps tool execution wit... | 256 |
| [do-truly-benefit-longer](skills/do-truly-benefit-longer/SKILL.md) | Optimize LLM context length for post-editing and refinement pipelines. Applies research showing that naively adding docu... | 252 |
| [domain-specific-knowledge-graphs-rag-enhanced](skills/domain-specific-knowledge-graphs-rag-enhanced/SKILL.md) | > | 217 |
| [dr-mas-stable-reinforcement-learning](skills/dr-mas-stable-reinforcement-learning/SKILL.md) | Design and implement stable reinforcement learning pipelines for multi-agent LLM systems using agent-wise advantage norm... | 203 |
| [draincode-stealthy-energy-consumption](skills/draincode-stealthy-energy-consumption/SKILL.md) | Evaluate and defend RAG-based code generation systems against energy-drain attacks that poison retrieval contexts to inf... | 221 |
| [drpg-decompose-retrieve-plan](skills/drpg-decompose-retrieve-plan/SKILL.md) | > | 253 |
| [dziribot-rag-intelligent-conversational](skills/dziribot-rag-intelligent-conversational/SKILL.md) | Build dialect-aware RAG conversational agents that handle non-standard orthography, code-switching, and multi-script inp... | 236 |
| [echo-open-research-platform](skills/echo-open-research-platform/SKILL.md) | Build and configure ECHO-style research platforms for running reproducible user studies comparing chat-based AI and web ... | 207 |
| [efficient-table-retrieval-understanding](skills/efficient-table-retrieval-understanding/SKILL.md) | | | 178 |
| [ema-policy-gradient-taming](skills/ema-policy-gradient-taming/SKILL.md) | Implement EMA Policy Gradient (EMA-PG) for stabilizing reinforcement learning fine-tuning of LLMs. Combines an Exponenti... | 228 |
| [empirical-mcts-continuous-agent-evolution](skills/empirical-mcts-continuous-agent-evolution/SKILL.md) | Applies Empirical-MCTS dual-loop reasoning: structured tree search with persistent memory that accumulates experience ac... | 195 |
| [evaluating-retrievalaugmented-generation-variants](skills/evaluating-retrievalaugmented-generation-variants/SKILL.md) | | | 223 |
| [evaluating-social-bias-rag](skills/evaluating-social-bias-rag/SKILL.md) | Evaluate and mitigate social bias in RAG pipelines. Use when: 'audit my RAG system for bias', 'check if retrieval introd... | 212 |
| [evaluation-oncotimia-system-supporting](skills/evaluation-oncotimia-system-supporting/SKILL.md) | Build RAG pipelines that transform unstructured clinical or domain-specific documents into structured form records using... | 213 |
| [evermembench-benchmarking-long-term-interactive](skills/evermembench-benchmarking-long-term-interactive/SKILL.md) | Build and evaluate long-term conversational memory systems for multi-party, multi-topic dialogues. Implements the EverMe... | 182 |
| [evolve-evolutionary-search-llm-based](skills/evolve-evolutionary-search-llm-based/SKILL.md) | Evolutionary search framework for LLM-driven Verilog/RTL generation and PPA optimization. Uses MCTS for functional corre... | 174 |
| [experience-driven-multi-agent-systems-training-fre](skills/experience-driven-multi-agent-systems-training-fre/SKILL.md) | Build self-evolving multi-agent systems that accumulate tool-level expertise through structured interaction without mode... | 168 |
| [farm-field-aware-resolution-intelligent](skills/farm-field-aware-resolution-intelligent/SKILL.md) | Build intelligent trigger-action automation systems using FARM's two-stage architecture: contrastive retrieval + multi-a... | 187 |
| [fin-rate-real-world-financial-analytics](skills/fin-rate-real-world-financial-analytics/SKILL.md) | Analyze SEC filings and financial disclosures using the Fin-RATE three-pathway methodology: detail-oriented reasoning wi... | 182 |
| [flyaoc-evaluating-agentic-ontology](skills/flyaoc-evaluating-agentic-ontology/SKILL.md) | Build multi-agent systems for end-to-end ontology curation from scientific literature. Applies FlyAOC's agent architectu... | 184 |
| [following-dragons-code-review-guided](skills/following-dragons-code-review-guided/SKILL.md) | Extract security-relevant signals from code review comments and translate them into fuzzer-guiding annotations using the... | 158 |
| [from-passive-metric-active](skills/from-passive-metric-active/SKILL.md) | Build systems that use LLM uncertainty as an active control signal -- routing computation, triggering tool calls, enabli... | 268 |
| [from-pragmas-partners-symbiotic](skills/from-pragmas-partners-symbiotic/SKILL.md) | Agentic High-Level Synthesis (HLS) optimization: autonomously analyze, insert, and tune C/C++ HLS pragmas (pipeline, unr... | 186 |
| [fs-researcher-test-time-scaling-long-horizon](skills/fs-researcher-test-time-scaling-long-horizon/SKILL.md) | > | 221 |
| [fullstack-agent-enhancing-agentic-fullstack](skills/fullstack-agent-enhancing-agentic-fullstack/SKILL.md) | Build production-grade full-stack web applications using a three-agent pipeline (Planning, Backend, Frontend) with devel... | 146 |
| [geogr-generative-retrieval-framework](skills/geogr-generative-retrieval-framework/SKILL.md) | Build spatio-temporal POI recommendation systems using generative retrieval with geo-aware semantic IDs and LLM alignmen... | 171 |
| [graph-anchored-knowledge-indexing-retrieval-augmen](skills/graph-anchored-knowledge-indexing-retrieval-augmen/SKILL.md) | Build iterative RAG pipelines that construct evolving knowledge graphs to anchor retrieval across multiple hops. Use whe... | 221 |
| [graph-based-agent-memory-taxonomy](skills/graph-based-agent-memory-taxonomy/SKILL.md) | Design and implement graph-based memory systems for LLM agents following the extraction-storage-retrieval-evolution life... | 279 |
| [graphagents-knowledge-graph-guided-agentic](skills/graphagents-knowledge-graph-guided-agentic/SKILL.md) | Build multi-agent pipelines that use knowledge graphs to guide LLM reasoning across domains. Agents specialize in proble... | 185 |
| [graphdancer-training-explore-reason](skills/graphdancer-training-explore-reason/SKILL.md) | Build agentic graph-exploration systems where an LLM navigates heterogeneous knowledge graphs through interleaved reason... | 243 |
| [greprag-empirical-study-optimization](skills/greprag-empirical-study-optimization/SKILL.md) | Lightweight, index-free repository-level code retrieval using ripgrep for context-aware code completion. Uses LLM-genera... | 190 |
| [harnessing-precision-querying-retrieval-augmented](skills/harnessing-precision-querying-retrieval-augmented/SKILL.md) | LLM-driven precision querying of structured tabular data via Python/Pandas code generation and retrieval-augmented extra... | 163 |
| [high-fidelity-textual-user](skills/high-fidelity-textual-user/SKILL.md) | Build RL-optimized unified textual user representations from heterogeneous data sources (profiles, activity logs, search... | 227 |
| [how-much-reasoning-retrieval-augmented](skills/how-much-reasoning-retrieval-augmented/SKILL.md) | Build contamination-aware hybrid RAG evaluation pipelines that couple knowledge graphs with text retrieval for multi-hop... | 178 |
| [how-personalized-memory-shape](skills/how-personalized-memory-shape/SKILL.md) | Rational preference utilization for personalized LLM assistants. Implements RP-Reasoner's pragmatic reasoning to selecti... | 216 |
| [hugrag-hierarchical-causal-knowledge](skills/hugrag-hierarchical-causal-knowledge/SKILL.md) | Build hierarchical causal knowledge graphs for RAG pipelines that suppress spurious correlations and enable cross-docume... | 168 |
| [identifying-adversary-tactics-techniques](skills/identifying-adversary-tactics-techniques/SKILL.md) | Identify MITRE ATT&CK Tactics, Techniques, and Procedures (TTPs) in decompiled malware binaries using the TTPDetect meth... | 198 |
| [iesr-mcts-based-modular-reasoning](skills/iesr-mcts-based-modular-reasoning/SKILL.md) | Convert natural language questions into SQL queries using MCTS-based modular reasoning inspired by the IESR framework. D... | 242 |
| [improving-user-privacy-personalized](skills/improving-user-privacy-personalized/SKILL.md) | Build privacy-preserving personalized LLM systems using the P³ (Private Personalized Prediction) client-server architect... | 192 |
| [internalizing-multi-agent-reasoning-accurate](skills/internalizing-multi-agent-reasoning-accurate/SKILL.md) | Distill multi-agent reasoning into a single efficient model for recommendation or retrieval. Use when: 'build a recommen... | 174 |
| [jade-bridging-strategic-operational-gap](skills/jade-bridging-strategic-operational-gap/SKILL.md) | Build jointly-optimized agentic RAG pipelines using the JADE pattern: a central planner co-adapted with specialized exec... | 248 |
| [jetformer-scalable-transformer-jet](skills/jetformer-scalable-transformer-jet/SKILL.md) | Build, train, compress, and deploy JetFormer encoder-only Transformers for particle jet tagging -- from offline analysis... | 167 |
| [koral-knowledge-graph-guided](skills/koral-knowledge-graph-guided/SKILL.md) | Build Knowledge Graph-guided LLM reasoning pipelines for operational telemetry analysis. Combines a Literature KG (extra... | 247 |
| [large-model-powered-evolutionary-code](skills/large-model-powered-evolutionary-code/SKILL.md) | Iteratively optimize code performance using LLM-driven evolutionary search on a phylogenetic tree. Applies PhyloEvolve-s... | 186 |
| [large-reasoning-failures](skills/large-reasoning-failures/SKILL.md) | Detect and mitigate known LLM reasoning failures during code generation, review, and problem-solving. Applies the taxono... | 218 |
| [large-scale-multidimensional-knowledge-profiling](skills/large-scale-multidimensional-knowledge-profiling/SKILL.md) | Build multidimensional profiling pipelines for large scientific paper corpora. Combines BERTopic clustering, LLM-structu... | 246 |
| [latent-chain-of-thought-as-planning](skills/latent-chain-of-thought-as-planning/SKILL.md) | Decouple reasoning from verbalization using PLaT-inspired latent planning. Maintains a broad solution space through para... | 200 |
| [learning-irrecoverable-error-localized-policy](skills/learning-irrecoverable-error-localized-policy/SKILL.md) | Debug multi-step tool-using agent pipelines by localizing the first irrecoverable error via binary-search rollback, then... | 175 |
| [learning-rate-matters-vanilla](skills/learning-rate-matters-vanilla/SKILL.md) | Configure optimal learning rates for LoRA fine-tuning of LLMs. Generates hyperparameter search configs, training scripts... | 203 |
| [legalmalr-multi-agent-query-understanding](skills/legalmalr-multi-agent-query-understanding/SKILL.md) | Multi-agent query reformulation and LLM reranking for retrieval over legal, regulatory, or domain-specific corpora. Use ... | 168 |
| [lemur-corpus-robust-fine-tuning](skills/lemur-corpus-robust-fine-tuning/SKILL.md) | Build multilingual legal document retrieval systems by fine-tuning embedding models on domain-specific corpora with cont... | 234 |
| [less-enough-synthesizing-diverse](skills/less-enough-synthesizing-diverse/SKILL.md) | Synthesize maximally diverse training data for LLM post-training using Feature Activation Coverage (FAC). Identifies mis... | 181 |
| [less-finetuning-retrieval-rethinking](skills/less-finetuning-retrieval-rethinking/SKILL.md) | | | 269 |
| [leveraging-data-say-no](skills/leveraging-data-say-no/SKILL.md) | Implement memory-augmented selective prediction for vision-language models using retrieval-based confidence scoring and ... | 194 |
| [leveraging-turkish-skill-extraction](skills/leveraging-turkish-skill-extraction/SKILL.md) | Extract and normalize skills from job postings using a two-stage LLM pipeline: dynamic few-shot skill identification fol... | 198 |
| [linear-merging-unlocks-simple](skills/linear-merging-unlocks-simple/SKILL.md) | Use linear model merging as a cheap proxy for data mixture optimization (DMO) in multimodal LLM fine-tuning. Instead of ... | 168 |
| [live-evo-online-evolution-agentic](skills/live-evo-online-evolution-agentic/SKILL.md) | Implement online self-evolving memory for LLM agents using dual-bank architecture (Experience Bank + Meta-Guideline Bank... | 204 |
| [livibench-omnimodal-benchmark-interactive](skills/livibench-omnimodal-benchmark-interactive/SKILL.md) | Build omnimodal benchmarks and evaluation pipelines for interactive video understanding (livestreams, real-time comments... | 238 |
| [llm-autodp-automatic-data-processing](skills/llm-autodp-automatic-data-processing/SKILL.md) | Automatically generate and optimize data processing pipelines for LLM fine-tuning datasets using an agent-driven iterati... | 168 |
| [locomo-plus-beyond-factual-cognitive-memory](skills/locomo-plus-beyond-factual-cognitive-memory/SKILL.md) | Build and evaluate cognitive memory systems for LLM dialogue agents that retain implicit user constraints (state, goals,... | 216 |
| [lycheedecode-accelerating-long-context-inference](skills/lycheedecode-accelerating-long-context-inference/SKILL.md) | Accelerate long-context LLM inference using hybrid-head sparse decoding with HardKuma-based head partitioning. Implement... | 179 |
| [made-benchmark-environments-closed-loop](skills/made-benchmark-environments-closed-loop/SKILL.md) | Build closed-loop discovery benchmarks where an agent iteratively proposes, evaluates, and refines candidates under a fi... | 144 |
| [magellan-autonomous-discovery-compiler](skills/magellan-autonomous-discovery-compiler/SKILL.md) | Evolve compiler optimization heuristics by coupling LLM code generation with evolutionary search and autotuning. Synthes... | 158 |
| [marti-mars2-scaling-multi-agent-self-search-reinfo](skills/marti-mars2-scaling-multi-agent-self-search-reinfo/SKILL.md) | Multi-agent tree-search code generation using heterogeneous agent collaboration with error-feedback refinement. Spawns m... | 204 |
| [martingale-foresight-sampling-principled](skills/martingale-foresight-sampling-principled/SKILL.md) | Implement Martingale Foresight Sampling (MFS) for principled LLM decoding with lookahead search. Replaces heuristic beam... | 220 |
| [medbeads-agent-native-immutable-data](skills/medbeads-agent-native-immutable-data/SKILL.md) | Build immutable, agent-native medical data pipelines using Merkle DAG structures (MedBeads pattern). Converts mutable EM... | 182 |
| [medspeak-knowledge-graph-aided-asr](skills/medspeak-knowledge-graph-aided-asr/SKILL.md) | Build knowledge-graph-aided ASR error correction pipelines for medical speech, using phonetic similarity + semantic retr... | 262 |
| [memadapter-fast-alignment-across](skills/memadapter-fast-alignment-across/SKILL.md) | Unify heterogeneous agent memory systems (explicit graphs, parametric weights, latent KV-caches) via generative subgraph... | 166 |
| [memcast-memory-driven-time-series](skills/memcast-memory-driven-time-series/SKILL.md) | Build memory-augmented time series forecasting systems using hierarchical experience storage (historical patterns, reaso... | 196 |
| [mermaid-memory-enhanced-retrieval-reasoning](skills/mermaid-memory-enhanced-retrieval-reasoning/SKILL.md) | Memory-enhanced multi-agent retrieval and reasoning for veracity assessment and fact-checking. Use when: 'verify this cl... | 189 |
| [mirror-multi-agent-framework-iterative](skills/mirror-multi-agent-framework-iterative/SKILL.md) | Translate natural language optimization problems into mathematical models and solver code using MIRROR's multi-agent pip... | 166 |
| [modality-gap-driven-subspace-alignment](skills/modality-gap-driven-subspace-alignment/SKILL.md) | Align multimodal embeddings (vision-language) by correcting the modality gap using the ReAlign/ReVision technique. Fixes... | 231 |
| [monte-carlo-tree-search](skills/monte-carlo-tree-search/SKILL.md) | > | 186 |
| [mpib-benchmark-medical-prompt](skills/mpib-benchmark-medical-prompt/SKILL.md) | Evaluate and defend clinical LLM systems against prompt injection attacks using the MPIB benchmark methodology. Implemen... | 177 |
| [mrag-benchmarking-retrieval-augmented-generation](skills/mrag-benchmarking-retrieval-augmented-generation/SKILL.md) | Build and evaluate biomedical RAG pipelines using the MRAG benchmark methodology. Configures retrieval, prompting, and g... | 183 |
| [muco-multi-turn-contrastive-learning](skills/muco-multi-turn-contrastive-learning/SKILL.md) | Implement multi-turn contrastive learning for multimodal embedding models. Restructures query-target pairs as multi-turn... | 260 |
| [multi-agent-teams-hold-experts](skills/multi-agent-teams-hold-experts/SKILL.md) | Prevent expertise dilution in multi-agent LLM workflows by applying findings from 'Multi-Agent Teams Hold Experts Back' ... | 151 |
| [multi-field-tool-retrieval](skills/multi-field-tool-retrieval/SKILL.md) | Implement multi-field tool retrieval systems that decompose tool documentation into structured fields (description, para... | 284 |
| [mulvul-retrieval-augmented-multi-agent-code](skills/mulvul-retrieval-augmented-multi-agent-code/SKILL.md) | Multi-agent vulnerability detection using coarse-to-fine routing, contrastive retrieval, and cross-model prompt evolutio... | 204 |
| [next-gen-captchas-leveraging-cognitive](skills/next-gen-captchas-leveraging-cognitive/SKILL.md) | Design and implement AI-resistant CAPTCHA systems that exploit the cognitive gap between humans and GUI agents. Covers p... | 253 |
| [omnirag-agent-agentic-omnimodal-reasoning](skills/omnirag-agent-agentic-omnimodal-reasoning/SKILL.md) | Build agentic multimodal RAG pipelines that answer questions over long audio-video content under resource constraints. U... | 278 |
| [on-use-support-conduction](skills/on-use-support-conduction/SKILL.md) | LLM-assisted systematic literature review and mapping study pipeline. Automates screening, data extraction, and classifi... | 167 |
| [one-size-many-fits](skills/one-size-many-fits/SKILL.md) | Build group-aware advertising image generation systems that align diverse user-segment click preferences instead of opti... | 236 |
| [optimizing-prompts-causal-approach](skills/optimizing-prompts-causal-approach/SKILL.md) | Optimize LLM prompts using causal inference (CPO). Isolates true prompt effectiveness from query difficulty via Double M... | 156 |
| [paperbanana-automating-academic-illustration](skills/paperbanana-automating-academic-illustration/SKILL.md) | Generate publication-ready academic illustrations using a multi-agent pipeline inspired by PaperBanana. Orchestrates ret... | 197 |
| [papersearchqa-learning-search-reason](skills/papersearchqa-learning-search-reason/SKILL.md) | Build iterative search-and-reason agents for scientific literature QA. Uses the PaperSearchQA pattern: interleaved think... | 233 |
| [pathwise-planning-world-automated](skills/pathwise-planning-world-automated/SKILL.md) | Multi-agent heuristic design framework that uses an entailment graph, policy/world-model/critic agents, and routed refle... | 165 |
| [phenolip-integrating-phenotype-ontology](skills/phenolip-integrating-phenotype-ontology/SKILL.md) | Build phenotype-aware medical vision-language models by integrating ontology knowledge graphs into CLIP-style pretrainin... | 234 |
| [polarmem-training-free-polarized-latent](skills/polarmem-training-free-polarized-latent/SKILL.md) | Build polarized memory systems for multimodal agents that encode both positive and negative evidence as graph constraint... | 183 |
| [precise-reducing-bias-evaluations](skills/precise-reducing-bias-evaluations/SKILL.md) | Implement the PRECISE framework to debias LLM-as-judge evaluations of search, ranking, and RAG systems by combining a sm... | 212 |
| [predicting-improving-test-time-scaling](skills/predicting-improving-test-time-scaling/SKILL.md) | Implement Scaling-Law Guided (SLG) Search for test-time compute optimization. Uses reward tail distribution estimation (... | 209 |
| [prograph-r1-progress-aware-reinforcement-learning](skills/prograph-r1-progress-aware-reinforcement-learning/SKILL.md) | Build progress-aware GraphRAG agents that traverse knowledge graphs with structure-aware hypergraph retrieval and dense ... | 167 |
| [prorag-process-supervised-reinforcement-learning](skills/prorag-process-supervised-reinforcement-learning/SKILL.md) | > | 157 |
| [protean-compiler-agile-framework](skills/protean-compiler-agile-framework/SKILL.md) | Guide fine-grained LLVM compiler phase ordering using the Protean framework's agile optimization approach — clustering p... | 149 |
| [pull-requests-as-training](skills/pull-requests-as-training/SKILL.md) | Apply the Clean-PR agentless repo-level code editing protocol: decompose issues into file localization, fine-grained nav... | 199 |
| [quasar-universal-autonomous-system](skills/quasar-universal-autonomous-system/SKILL.md) | Build autonomous multi-scale scientific simulation pipelines using the QUASAR architecture: a Strategist-Operator-Evalua... | 165 |
| [raca-representation-aware-coverage-criteria](skills/raca-representation-aware-coverage-criteria/SKILL.md) | Evaluate and improve LLM safety test suites using representation-aware coverage criteria. Implements the RACA framework ... | 242 |
| [ragturk-best-practices-retrieval](skills/ragturk-best-practices-retrieval/SKILL.md) | Design and optimize RAG pipelines for Turkish and other morphologically rich languages (Turkish, Finnish, Hungarian, Kor... | 208 |
| [raicl-retrieval-augmented-in-context-learning](skills/raicl-retrieval-augmented-in-context-learning/SKILL.md) | Build retrieval-augmented in-context learning (RAICL) pipelines that convert time-series or signal data into images and ... | 228 |
| [reasoning-augmented-representations-multimodal-ret](skills/reasoning-augmented-representations-multimodal-ret/SKILL.md) | Decouple reasoning from embedding compression in multimodal retrieval pipelines by enriching queries and corpus entries ... | 223 |
| [rebel-hidden-knowledge-recovery](skills/rebel-hidden-knowledge-recovery/SKILL.md) | Machine unlearning for LLMs aims to remove sensitive or copyrighted data from trained models. Implements techniques from... | 28 |
| [redvisor-reasoning-aware-prompt-injection](skills/redvisor-reasoning-aware-prompt-injection/SKILL.md) | Defend LLM applications against prompt injection using RedVisor's two-phase reasoning-then-responding architecture. Impl... | 223 |
| [report-nsf-workshop-ai](skills/report-nsf-workshop-ai/SKILL.md) | Apply AI techniques from the NSF AI-for-EDA workshop to hardware design tasks: RTL code generation from natural language... | 276 |
| [research-multi-stage-machine-learning](skills/research-multi-stage-machine-learning/SKILL.md) | Build multi-stage search pipelines that separate recall from precision for discovering datasets, documents, or resources... | 279 |
| [rethinking-reranker-boundary-aware-evidence](skills/rethinking-reranker-boundary-aware-evidence/SKILL.md) | > | 159 |
| [rpo-rag-aligning-small-relation-aware](skills/rpo-rag-aligning-small-relation-aware/SKILL.md) | Build knowledge-graph-grounded RAG pipelines that align small LLMs (under 8B params) with relation-aware preference opti... | 259 |
| [sar-rag-atr-visual-question](skills/sar-rag-atr-visual-question/SKILL.md) | | | 378 |
| [scidatacopilot-agentic-data-preparation](skills/scidatacopilot-agentic-data-preparation/SKILL.md) | Build agentic pipelines that ingest heterogeneous raw scientific data, parse research intent, and produce analysis-ready... | 240 |
| [sdr-cir-semantic-debias-retrieval](skills/sdr-cir-semantic-debias-retrieval/SKILL.md) | Build training-free composed image retrieval systems that combine a reference image with modification text to find targe... | 171 |
| [sed-sft-selectively-encouraging-diversity](skills/sed-sft-selectively-encouraging-diversity/SKILL.md) | Implement SED-SFT selective entropy regularization to combat mode collapse in LLM supervised fine-tuning. Use when: 'add... | 229 |
| [self-evolving-recommendation-system-end-to-end](skills/self-evolving-recommendation-system-end-to-end/SKILL.md) | Build autonomous ML optimization pipelines that use LLM agents to generate, evaluate, and deploy model improvements in a... | 168 |
| [shardmemo-masked-moe-routing](skills/shardmemo-masked-moe-routing/SKILL.md) | Implement ShardMemo-style tiered, sharded memory with masked Mixture-of-Experts routing for agentic LLM systems. Use whe... | 176 |
| [shopsimulator-evaluating-exploring-rl-driven](skills/shopsimulator-evaluating-exploring-rl-driven/SKILL.md) | Build and evaluate LLM-based shopping assistant agents using structured multi-turn dialogue, personalized product search... | 285 |
| [snapmla-efficient-longcontext-mla](skills/snapmla-efficient-longcontext-mla/SKILL.md) | While FP8 attention has shown substantial promise in innovations like FlashAttention-3, its integration into the decodin... | 88 |
| [sparc-rag-adaptive-sequential-parallel-scaling](skills/sparc-rag-adaptive-sequential-parallel-scaling/SKILL.md) | Implement multi-agent RAG systems with coordinated sequential-parallel scaling and shared context management for complex... | 248 |
| [sqlagent-learning-explore-before](skills/sqlagent-learning-explore-before/SKILL.md) | Explore unfamiliar databases before writing SQL by building a local knowledge base of schema fragments, executable queri... | 201 |
| [steuerllm-local-specialized-german](skills/steuerllm-local-specialized-german/SKILL.md) | Build domain-specialized LLM pipelines for formal-rule domains (tax law, legal, regulatory) using retrieval-augmented sy... | 203 |
| [strong-reasoning-isnt-enough](skills/strong-reasoning-isnt-enough/SKILL.md) | Build interactive diagnostic agents that systematically elicit evidence before concluding, using the REFINE (Reasoning-E... | 251 |
| [supporting-software-engineering-tasks](skills/supporting-software-engineering-tasks/SKILL.md) | Generate test scenarios from requirements and retrieve/analyze software engineering documents using a supervisor-worker ... | 169 |
| [swe-context-bench-benchmark](skills/swe-context-bench-benchmark/SKILL.md) | Reuse prior coding experience across related repository tasks. Accumulate, summarize, retrieve, and inject compact exper... | 180 |
| [synthesizing-file-level-data-unit](skills/synthesizing-file-level-data-unit/SKILL.md) | Generate high-quality unit tests with self-debugging repair loops and chain-of-thought reasoning. Produces tests with me... | 222 |
| [table-as-search-formulate-long-horizon-agentic](skills/table-as-search-formulate-long-horizon-agentic/SKILL.md) | Structured table-completion framework for long-horizon information seeking. Converts complex research queries into datab... | 203 |
| [temp-r1-unified-autonomous-agent](skills/temp-r1-unified-autonomous-agent/SKILL.md) | Build autonomous agents that answer complex temporal questions over knowledge graphs or time-stamped datasets using stru... | 200 |
| [thinking-broad-acting-fast](skills/thinking-broad-acting-fast/SKILL.md) | Build production search relevance systems using Multi-Perspective Chain-of-Thought distillation into lightweight student... | 210 |
| [toolweaver-weaving-collaborative-semantics](skills/toolweaver-weaving-collaborative-semantics/SKILL.md) | Design scalable tool retrieval systems using hierarchical code tokenization that captures collaborative tool semantics. ... | 181 |
| [towards-ai-evaluation-domain-specific](skills/towards-ai-evaluation-domain-specific/SKILL.md) | Build and evaluate domain-specific RAG systems with iterative user-feedback refinement, source grounding, and structured... | 260 |
| [towards-autonomous-mathematics-research](skills/towards-autonomous-mathematics-research/SKILL.md) | Iterative generate-verify-revise agent for mathematical research problems. Implements the Aletheia loop: decompose a har... | 239 |
| [towards-understanding-best-practices](skills/towards-understanding-best-practices/SKILL.md) | Quantize vision-language models (VLMs) component-by-component using optimal bit-width strategies derived from multimodal... | 189 |
| [tracellm-leveraging-prompt-engineering](skills/tracellm-leveraging-prompt-engineering/SKILL.md) | Establish and verify traceability links between software artifacts (requirements, design docs, test cases, regulations) ... | 179 |
| [training-multi-turn-search-agent](skills/training-multi-turn-search-agent/SKILL.md) | Build and train multi-turn search agents using BranPO (Branching Relative Policy Optimization) with contrastive dynamic ... | 177 |
| [tre-encouraging-exploration-trust](skills/tre-encouraging-exploration-trust/SKILL.md) | Implement Trust Region Entropy (TRE) for LLM reinforcement learning. Constrains entropy-based exploration to plausible t... | 204 |
| [tutorial-reasoning-ir-ir](skills/tutorial-reasoning-ir-ir/SKILL.md) | Build reasoning-enhanced information retrieval pipelines that go beyond semantic matching. Applies five methodological f... | 249 |
| [unifying-ranking-generation-query](skills/unifying-ranking-generation-query/SKILL.md) | Build production RAG-based query auto-completion systems that unify ranking and generation with multi-objective alignmen... | 158 |
| [use-graph-it-needs](skills/use-graph-it-needs/SKILL.md) | Implement adaptive RAG pipelines that route queries to dense retrieval, graph-based retrieval, or a weighted fusion base... | 254 |
| [videothinker-building-agentic-videollms](skills/videothinker-building-agentic-videollms/SKILL.md) | Build agentic video understanding systems with LLM-guided tool reasoning. Implements the VideoThinker pattern: confidenc... | 204 |
| [vidvec-unlocking-video-mllm](skills/vidvec-unlocking-video-mllm/SKILL.md) | Extract high-quality video-text embeddings from generative MLLMs using intermediate-layer representations and text-only ... | 197 |
| [vihermes-graph-grounded-multihop-question](skills/vihermes-graph-grounded-multihop-question/SKILL.md) | Build graph-grounded multihop QA systems over regulatory and hierarchically structured documents. Combines vector simila... | 264 |
| [viola-video-in-context-learning](skills/viola-video-in-context-learning/SKILL.md) | Apply the VIOLA framework for label-efficient in-context learning on video or multimodal data. Uses density-uncertainty-... | 221 |
| [vision-deepresearch-benchmark-rethinking-visual-te](skills/vision-deepresearch-benchmark-rethinking-visual-te/SKILL.md) | Build and evaluate Vision-DeepResearch pipelines that combine cropped visual search with multi-hop textual search for ro... | 216 |
| [vision-deepresearch-incentivizing-deepresearch-cap](skills/vision-deepresearch-incentivizing-deepresearch-cap/SKILL.md) | Multi-turn, multi-entity, multi-scale visual and textual deep research agent for answering complex questions about image... | 180 |
| [vlm-guided-iterative-refinement-surgical](skills/vlm-guided-iterative-refinement-surgical/SKILL.md) | Build iterative VLM-guided refinement pipelines for image segmentation tasks, especially surgical/medical imagery. Uses ... | 255 |
| [wavlink-compact-audio-text-embeddings](skills/wavlink-compact-audio-text-embeddings/SKILL.md) | Build compact audio-text embedding systems using WavLink's global Whisper token architecture with Matryoshka dimensional... | 197 |
| [wdscaling-parallel-tool-calling-deep](skills/wdscaling-parallel-tool-calling-deep/SKILL.md) | Scale deep research tasks by issuing parallel tool calls (width) alongside sequential reasoning (depth), following the W... | 161 |
| [what-should-cite-rag](skills/what-should-cite-rag/SKILL.md) | Build multi-level RAG pipelines for academic citation prediction and literature discovery. Use when the user asks to 'fi... | 193 |
| [when-agents-fail-comprehensive](skills/when-agents-fail-comprehensive/SKILL.md) | Diagnose and fix bugs in LLM agent systems using a research-backed taxonomy of 11 bug types, 9 root causes, and 12 obser... | 240 |
| [when-better-prompts-hurt](skills/when-better-prompts-hurt/SKILL.md) | Evaluation-driven prompt iteration using the Define-Test-Diagnose-Fix loop and Minimum Viable Evaluation Suite (MVES). P... | 196 |
| [when-iterative-rag-beats](skills/when-iterative-rag-beats/SKILL.md) | Build iterative retrieval-reasoning RAG pipelines that outperform single-shot retrieval, using staged evidence gathering... | 244 |
| [when-should-search-more](skills/when-should-search-more/SKILL.md) | > | 300 |
| [why-deep-research-agent](skills/why-deep-research-agent/SKILL.md) | > | 199 |
| [wideseek-r1-exploring-width-scaling](skills/wideseek-r1-exploring-width-scaling/SKILL.md) | Decompose broad information-seeking tasks into parallel subtasks using a lead-agent-subagent pattern with isolated conte... | 197 |
| [wiki-live-challenge-challenging](skills/wiki-live-challenge-challenging/SKILL.md) | Evaluate deep research agents and LLM-generated long-form articles using the Wiki Live Challenge framework: 39 fine-grai... | 264 |
| [yunque-deepresearch-technical-report](skills/yunque-deepresearch-technical-report/SKILL.md) | Hierarchical multi-agent deep research framework with dynamic context management and supervisor-based error recovery. Im... | 191 |
| [zero2text-zero-training-cross-domain-inversion](skills/zero2text-zero-training-cross-domain-inversion/SKILL.md) | Implement embedding inversion attacks that reconstruct original text from vector embeddings without training data, using... | 188 |

---

## Fine-tuning & Training

**191 skills**

| Skill | Description | Lines |
|-------|-------------|-------|
| [1100-high-efficiency-visual-adapter-complex](skills/1100-high-efficiency-visual-adapter-complex/SKILL.md) | Implement CoLin (Complex Linear Projection) adapters for parameter-efficient fine-tuning of vision foundation models. Ad... | 251 |
| [acegrpo-adaptive-curriculum-group](skills/acegrpo-adaptive-curriculum-group/SKILL.md) | Adaptive curriculum-driven iterative optimization for autonomous ML engineering tasks. Uses Evolving Data Buffers and Le... | 220 |
| [agentark-distilling-multi-agent-intelligence](skills/agentark-distilling-multi-agent-intelligence/SKILL.md) | Distill multi-agent debate reasoning into a single LLM's behavior. Apply AgentArk's three-tier distillation strategy to ... | 199 |
| [agentic-reinforcement-learning-empowers](skills/agentic-reinforcement-learning-empowers/SKILL.md) | Build tool-augmented agent systems that decouple domain reasoning from knowledge storage, following the ChemCRAFT patter... | 242 |
| [agentskiller-scaling-generalist-agent](skills/agentskiller-scaling-generalist-agent/SKILL.md) | Synthesize multi-turn agent interaction data across semantically linked domains using DAG-based pipelines, domain ontolo... | 178 |
| [aiano-enhancing-information-retrieval](skills/aiano-enhancing-information-retrieval/SKILL.md) | Build AI-augmented annotation pipelines for creating high-quality information retrieval and QA datasets. Combines LLM-ge... | 161 |
| [aligntune-modular-toolkit-post-training](skills/aligntune-modular-toolkit-post-training/SKILL.md) | > | 287 |
| [amem4rec-leveraging-cross-user-similarity](skills/amem4rec-leveraging-cross-user-similarity/SKILL.md) | Build agentic recommendation systems that learn collaborative filtering signals through cross-user memory evolution -- n... | 273 |
| [asa-training-free-representation-engineering](skills/asa-training-free-representation-engineering/SKILL.md) | Implement Activation Steering Adapter (ASA) for training-free tool-calling improvement in LLM agents. Use when: 'fix laz... | 170 |
| [assessment-generative-named-entity](skills/assessment-generative-named-entity/SKILL.md) | Build generative NER systems using LLMs with optimal output formats and prompt engineering. Use when: 'extract entities ... | 245 |
| [automated-customization-enterprise-code](skills/automated-customization-enterprise-code/SKILL.md) | Customize LLMs for enterprise code repositories using semantic scopes -- automatically partition codebases into meaningf... | 153 |
| [autonomous-chain-of-thought-distillation-graph-bas](skills/autonomous-chain-of-thought-distillation-graph-bas/SKILL.md) | Implement FraudCoT-style graph-aware chain-of-thought distillation for fraud detection on text-attributed graphs. Combin... | 199 |
| [bear-beam-search-aware-optimization-recommendation](skills/bear-beam-search-aware-optimization-recommendation/SKILL.md) | Implement BEAR (Beam-SEarch-Aware Regularization) to fix training-inference mismatch in LLM-based recommendation systems... | 220 |
| [better-as-generators-than](skills/better-as-generators-than/SKILL.md) | Generate synthetic labeled datasets with LLMs to train smaller, cheaper classifiers -- especially for low-resource langu... | 178 |
| [better-generalizing-unseen-concepts](skills/better-generalizing-unseen-concepts/SKILL.md) | Build biomedical concept recognition systems that generalize to unseen ontology concepts using hierarchical indexing and... | 143 |
| [beyond-alignment-expanding-reasoning](skills/beyond-alignment-expanding-reasoning/SKILL.md) | Apply Manifold-Reshaping Policy Optimization (MRPO) to expand LLM reasoning capacity beyond alignment. Implements spectr... | 250 |
| [beyond-instrumental-substitutive-paradigms](skills/beyond-instrumental-substitutive-paradigms/SKILL.md) | Audit and diagnose cultural bias artifacts in LLM-powered applications using the Machine Culture framework. Detects Cult... | 189 |
| [beyond-superficial-unlearning-sharpness-aware](skills/beyond-superficial-unlearning-sharpness-aware/SKILL.md) | Implement sharpness-aware robust erasure (SARE) for hallucination removal in multimodal LLMs. Uses Targeted-SAM to flatt... | 269 |
| [bi-directional-bias-attribution-debiasing](skills/bi-directional-bias-attribution-debiasing/SKILL.md) | Detect and mitigate social biases in LLM outputs using neuron-level attribution and intervention, without modifying prom... | 184 |
| [binaryppo-policy-optimization-binary](skills/binaryppo-policy-optimization-binary/SKILL.md) | Implement BinaryPPO, an offline RL framework that reformulates binary classification as reward maximization with confide... | 173 |
| [bridging-modality-gap-roadside](skills/bridging-modality-gap-roadside/SKILL.md) | Build training-free pipelines that convert sparse 3D LiDAR point clouds into depth-encoded 2D images for classification ... | 211 |
| [can-post-training-transform-causal](skills/can-post-training-transform-causal/SKILL.md) | Perform rigorous causal inference tasks using structured reasoning pipelines inspired by CauGym. Estimate treatment effe... | 179 |
| [causaltad-injecting-causal-knowledge](skills/causaltad-injecting-causal-knowledge/SKILL.md) | Detect anomalies in tabular data by injecting causal column relationships into LLM-based detection pipelines. Reorders a... | 267 |
| [cgpt-cluster-guided-partial-tables](skills/cgpt-cluster-guided-partial-tables/SKILL.md) | Improve table retrieval by constructing cluster-guided partial tables and generating synthetic training queries with LLM... | 230 |
| [cimrag-cim-aware-domain-adaptive-noise-resilient](skills/cimrag-cim-aware-domain-adaptive-noise-resilient/SKILL.md) | Build noise-resilient RAG retrieval pipelines for edge/resource-constrained deployments. Implements TONEL (Task-Oriented... | 227 |
| [compar-ia-french-governments](skills/compar-ia-french-governments/SKILL.md) | Build multilingual LLM evaluation arenas and preference data collection pipelines modeled on France's compar:IA platform... | 310 |
| [cope-clipped-rope-as](skills/cope-clipped-rope-as/SKILL.md) | Implement CoPE (Clipped RoPE) soft clipping of low-frequency rotary positional embedding components to extend LLM contex... | 215 |
| [cord-bridging-audio-text-reasoning](skills/cord-bridging-audio-text-reasoning/SKILL.md) | Implement CORD (Cross-modal On-policy Distillation) to bridge modality gaps in multimodal AI systems. Applies weighted s... | 238 |
| [curate-train-refine-closed-loop-agentic-framework-](skills/curate-train-refine-closed-loop-agentic-framework/SKILL.md) | Build lightweight text classifiers from zero labeled data using an agentic Curate-Train-Refine loop. An LLM generates sy... | 148 |
| [cure-curriculum-guided-multi-task-training](skills/cure-curriculum-guided-multi-task-training/SKILL.md) | Implement error-aware curriculum learning for multi-task training pipelines, especially medical/vision-language models. ... | 234 |
| [d-orca-dialogue-centric-optimization-robust](skills/d-orca-dialogue-centric-optimization-robust/SKILL.md) | Build dialogue-centric audio-visual captioning pipelines that identify who spoke what and when in multi-party video conv... | 228 |
| [d2quant-accurate-low-bit-post-training-weight](skills/d2quant-accurate-low-bit-post-training-weight/SKILL.md) | Apply the D2Quant post-training weight quantization framework to compress LLMs to sub-4-bit precision (2-bit, 3-bit) wit... | 229 |
| [darwin-dynamic-agentically-rewriting](skills/darwin-dynamic-agentically-rewriting/SKILL.md) | Evolutionary multi-agent code optimization using genetic algorithms. Agents mutate each other's training/configuration c... | 167 |
| [data-centric-interpretability-llm-based-multi-agen](skills/data-centric-interpretability-llm-based-multi-agen/SKILL.md) | Analyze LLM agent behavior across training runs using Sparse Autoencoder (SAE) features and LLM-summarizer pipelines. Gr... | 189 |
| [datachef-cooking-up-optimal](skills/datachef-cooking-up-optimal/SKILL.md) | Automate data recipe generation for LLM fine-tuning and adaptation. Generates executable data processing pipelines (filt... | 207 |
| [davinci-dev-agent-native-mid-training-software](skills/davinci-dev-agent-native-mid-training-software/SKILL.md) | Apply daVinci-Dev's agent-native workflow to software engineering tasks: navigate repos, localize bugs, plan edits, appl... | 171 |
| [dcopilot-generative-ai-empowered-policy](skills/dcopilot-generative-ai-empowered-policy/SKILL.md) | Build hybrid LLM + hypernetwork systems that generate control policies for dynamic environments. Uses LLM-based reward s... | 297 |
| [diffa-2-practical-diffusion-general](skills/diffa-2-practical-diffusion-general/SKILL.md) | Design and implement diffusion-based large audio language models (LALMs) using the DIFFA-2 architecture — dual-adapter p... | 237 |
| [diffusion-lms-approximate-optimal](skills/diffusion-lms-approximate-optimal/SKILL.md) | Search and retrieve research on Calibrated Adaptive Length (CAL) for diffusion language model infilling. Helps users und... | 170 |
| [diffusion-pretrained-dense-contextual-embeddings](skills/diffusion-pretrained-dense-contextual-embeddings/SKILL.md) | Build production retrieval systems using pplx-embed, diffusion-pretrained dense and contextualized embedding models with... | 183 |
| [dispo-enhancing-training-efficiency](skills/dispo-enhancing-training-efficiency/SKILL.md) | Implement the DISPO reinforcement learning algorithm for training LLMs on mathematical reasoning with decoupled importan... | 277 |
| [distilling-reasoning-graph-concept](skills/distilling-reasoning-graph-concept/SKILL.md) | Distill LLM reasoning into a DAG of modular concept predictors for efficient, interpretable classification. Use when ask... | 170 |
| [dllm-searcher-adapting-diffusion-large](skills/dllm-searcher-adapting-diffusion-large/SKILL.md) | Implement the P-ReAct parallel reasoning-and-acting agent paradigm from DLLM-Searcher, which overlaps tool execution wit... | 256 |
| [domain-adaptation-synthetic-data-fine-tuning-germa](skills/domain-adaptation-synthetic-data-fine-tuning-germa/SKILL.md) | Generate difficulty-graded synthetic QA datasets from authoritative domain documents (laws, regulations, standards) and ... | 193 |
| [dr-mas-stable-reinforcement-learning](skills/dr-mas-stable-reinforcement-learning/SKILL.md) | Design and implement stable reinforcement learning pipelines for multi-agent LLM systems using agent-wise advantage norm... | 203 |
| [drugr-optimizing-molecular-drugs](skills/drugr-optimizing-molecular-drugs/SKILL.md) | Optimize molecular drug candidates using LLM-based explicit pharmacological reasoning over SMILES structures. Applies th... | 187 |
| [duogen-general-purpose-interleaved](skills/duogen-general-purpose-interleaved/SKILL.md) | Design and implement interleaved multimodal generation pipelines that alternate between text and image generation using ... | 209 |
| [dynaweb-model-based-reinforcement-learning](skills/dynaweb-model-based-reinforcement-learning/SKILL.md) | Build model-based RL training pipelines for web agents using learned world models (environment simulators) that predict ... | 193 |
| [ecg-r1-protocol-guided-modality-agnostic-mllm](skills/ecg-r1-protocol-guided-modality-agnostic-mllm/SKILL.md) | Build protocol-guided medical AI interpretation pipelines with structured diagnostic reasoning, modality-robust architec... | 260 |
| [efficient-estimation-kernel-surrogate](skills/efficient-estimation-kernel-surrogate/SKILL.md) | Build kernel surrogate models to attribute how individual training tasks influence a target task's performance, capturin... | 199 |
| [ema-policy-gradient-taming](skills/ema-policy-gradient-taming/SKILL.md) | Implement EMA Policy Gradient (EMA-PG) for stabilizing reinforcement learning fine-tuning of LLMs. Combines an Exponenti... | 228 |
| [emoshift-lightweight-activation-steering](skills/emoshift-lightweight-activation-steering/SKILL.md) | Implement lightweight activation steering for emotion-controllable speech synthesis. Adds learned steering vectors to LL... | 227 |
| [emotion-llamav2-mmeverse-framework-benchmark](skills/emotion-llamav2-mmeverse-framework-benchmark/SKILL.md) | Build multimodal emotion understanding systems using the Emotion-LLaMAv2 architecture and MMEVerse benchmark methodology... | 232 |
| [emotionthinker-prosody-aware-reinforcement-learnin](skills/emotionthinker-prosody-aware-reinforcement-learnin/SKILL.md) | Build prosody-aware speech emotion reasoning pipelines using Chain-of-Thought RL. Implements EmotionThinker's GRPO-PTR t... | 292 |
| [evolving-tool-user-creator](skills/evolving-tool-user-creator/SKILL.md) | Transform Claude from a static tool user into a dynamic tool creator using the UCT (User-to-Creator Transformation) fram... | 181 |
| [experience-driven-multi-agent-systems-training-fre](skills/experience-driven-multi-agent-systems-training-fre/SKILL.md) | Build self-evolving multi-agent systems that accumulate tool-level expertise through structured interaction without mode... | 168 |
| [explainable-deepfake-detection-rl](skills/explainable-deepfake-detection-rl/SKILL.md) | Build explainable deepfake detection systems using RL-enhanced Self-Blended Images and Chain-of-Thought reasoning. Use w... | 296 |
| [fast-slow-training-multimodal-visual](skills/fast-slow-training-multimodal-visual/SKILL.md) | Implement DualSpeed fast-slow training for multimodal LLMs with visual token pruning. Use when: 'speed up MLLM training'... | 230 |
| [fedkrso-communication-memory-federated](skills/fedkrso-communication-memory-federated/SKILL.md) | Implement FedKRSO (Federated K-Seed Random Subspace Optimization) for communication- and memory-efficient federated fine... | 214 |
| [fimi-domain-specific-indian-finance](skills/fimi-domain-specific-indian-finance/SKILL.md) | Build domain-specialized AI agents for Indian financial systems (UPI, NPCI, RBI) using multi-stage training pipeline pat... | 243 |
| [fine-tuning-gpt-5-gpu-kernel](skills/fine-tuning-gpt-5-gpu-kernel/SKILL.md) | Generate optimized GPU kernels in Triton from PyTorch reference code using the Makora RL-based iterative refinement work... | 259 |
| [flashvid-video-training-free-tree-based](skills/flashvid-video-training-free-tree-based/SKILL.md) | Accelerate Video Large Language Models (VLLMs) by compressing visual tokens using FlashVID's training-free spatiotempora... | 168 |
| [flexible-entropy-control-rlvr](skills/flexible-entropy-control-rlvr/SKILL.md) | Implement dynamic entropy control for RLVR/GRPO/PPO training of LLMs using gradient-preserving clipping with adaptive th... | 205 |
| [fnf-functional-network-fingerprint](skills/fnf-functional-network-fingerprint/SKILL.md) | Detect whether a suspect LLM is derived from a victim model using Functional Network Fingerprints (FNF). Applies neurosc... | 215 |
| [found-rl-foundation-model-enhanced-reinforcement](skills/found-rl-foundation-model-enhanced-reinforcement/SKILL.md) | Architect asynchronous VLM-enhanced RL training pipelines that decouple heavy foundation model inference from simulation... | 264 |
| [from-classification-ranking-enhancing](skills/from-classification-ranking-enhancing/SKILL.md) | Reframe subjective classification tasks as ranking problems with GRPO reinforcement learning. Use when building personal... | 178 |
| [from-data-behavior-predicting](skills/from-data-behavior-predicting/SKILL.md) | Predict unintended LLM behaviors (bias, safety regressions) from training data BEFORE fine-tuning, using the MDF (Manipu... | 209 |
| [from-passive-metric-active](skills/from-passive-metric-active/SKILL.md) | Build systems that use LLM uncertainty as an active control signal -- routing computation, triggering tool calls, enabli... | 268 |
| [from-utterance-vividity-training](skills/from-utterance-vividity-training/SKILL.md) | Train expressive subtitle translation LLMs using Adaptive Local Preference Optimization (ALPO) — a segment-level prefere... | 257 |
| [gametalk-training-strategic-conversation](skills/gametalk-training-strategic-conversation/SKILL.md) | Build multi-agent strategic conversation systems where LLMs negotiate, coordinate, and optimize long-term objectives thr... | 207 |
| [graphdancer-training-explore-reason](skills/graphdancer-training-explore-reason/SKILL.md) | Build agentic graph-exploration systems where an LLM navigates heterogeneous knowledge graphs through interleaved reason... | 243 |
| [group-distributionally-robust-optimization-driven](skills/group-distributionally-robust-optimization-driven/SKILL.md) | Apply Group Distributionally Robust Optimization (GDRO) to RL-based LLM training. Dynamically classify prompts by diffic... | 205 |
| [he-snr-uncovering-latent-logic](skills/he-snr-uncovering-latent-logic/SKILL.md) | Evaluate and optimize LLM training data quality for software engineering tasks using the HE-SNR (High-Entropy Signal-to-... | 234 |
| [how-decoder-only-perceive-users](skills/how-decoder-only-perceive-users/SKILL.md) | Implement Gradient-Guided Soft Masking (GGSM) attention strategies for adapting decoder-only LLMs to user representation... | 235 |
| [hqp-sensitivity-aware-hybrid-quantization](skills/hqp-sensitivity-aware-hybrid-quantization/SKILL.md) | Apply the HQP framework to compress and accelerate PyTorch models for edge deployment using sensitivity-aware structural... | 189 |
| [hyperoffload-graph-driven-hierarchical-memory](skills/hyperoffload-graph-driven-hierarchical-memory/SKILL.md) | Design and implement compiler-driven hierarchical memory offloading for LLM inference and training on multi-tier memory ... | 226 |
| [improve-systems-user-logs](skills/improve-systems-user-logs/SKILL.md) | Implement the UNO (User log-driveN Optimization) framework to improve LLM-powered systems by distilling user interaction... | 201 |
| [innovator-vl-multimodal-scientific-discovery](skills/innovator-vl-multimodal-scientific-discovery/SKILL.md) | Build data-efficient multimodal scientific ML pipelines using Innovator-VL's principled training methodology. Applies tr... | 247 |
| [internalizing-multi-agent-reasoning-accurate](skills/internalizing-multi-agent-reasoning-accurate/SKILL.md) | Distill multi-agent reasoning into a single efficient model for recommendation or retrieval. Use when: 'build a recommen... | 174 |
| [internalizing-reasoning-discovery-replay](skills/internalizing-reasoning-discovery-replay/SKILL.md) | Apply the STIR (Self-Distilled Tools for Internal Reasoning) pattern to build systems that discover reusable reasoning p... | 253 |
| [isd-agent-bench-comprehensive-benchmark-evaluating](skills/isd-agent-bench-comprehensive-benchmark-evaluating/SKILL.md) | Build and evaluate LLM-based Instructional Design agents using the ADDIE framework, Context Matrix scenario generation, ... | 210 |
| [joint-continual-learning-local](skills/joint-continual-learning-local/SKILL.md) | Implement DA-GRPO (Dual-Advantage Group Relative Policy Optimization) for jointly training local small language models w... | 270 |
| [just-in-time-reinforcement-learning-continual](skills/just-in-time-reinforcement-learning-continual/SKILL.md) | Implement JitRL-style continual learning for LLM agents: training-free policy optimization via experience memory, advant... | 203 |
| [knowledge-graphs-implicit-reward](skills/knowledge-graphs-implicit-reward/SKILL.md) | Build compositional reasoning systems that use knowledge graph paths as reward signals to ground LLM reasoning in verifi... | 168 |
| [layer-wise-lora-fine-tuning-similarity](skills/layer-wise-lora-fine-tuning-similarity/SKILL.md) | Selectively apply LoRA adapters to only the most important transformer layers using CKA similarity-based layer importanc... | 221 |
| [learning-rate-matters-vanilla](skills/learning-rate-matters-vanilla/SKILL.md) | Configure optimal learning rates for LoRA fine-tuning of LLMs. Generates hyperparameter search configs, training scripts... | 203 |
| [legalone-family-foundation-reliable](skills/legalone-family-foundation-reliable/SKILL.md) | Build domain-specialized LLM training pipelines using the LegalOne three-phase methodology: Plasticity-Adjusted Sampling... | 259 |
| [lemur-corpus-robust-fine-tuning](skills/lemur-corpus-robust-fine-tuning/SKILL.md) | Build multilingual legal document retrieval systems by fine-tuning embedding models on domain-specific corpora with cont... | 234 |
| [less-enough-synthesizing-diverse](skills/less-enough-synthesizing-diverse/SKILL.md) | Synthesize maximally diverse training data for LLM post-training using Feature Activation Coverage (FAC). Identifies mis... | 181 |
| [linear-merging-unlocks-simple](skills/linear-merging-unlocks-simple/SKILL.md) | Use linear model merging as a cheap proxy for data mixture optimization (DMO) in multimodal LLM fine-tuning. Instead of ... | 168 |
| [linguamap-which-layers-speak](skills/linguamap-which-layers-speak/SKILL.md) | Diagnose and fix multilingual LLM failures using layer-localized analysis and selective fine-tuning. Use when: 'my model... | 230 |
| [llm-autodp-automatic-data-processing](skills/llm-autodp-automatic-data-processing/SKILL.md) | Automatically generate and optimize data processing pipelines for LLM fine-tuning datasets using an agent-driven iterati... | 168 |
| [llm-enhanced-reinforcement-learning-long-term](skills/llm-enhanced-reinforcement-learning-long-term/SKILL.md) | Build hierarchical recommendation systems that combine LLM semantic planning with RL fine-grained optimization for long-... | 247 |
| [llm-not-all-you](skills/llm-not-all-you/SKILL.md) | Systematic model selection advisor for classification tasks — chooses between classical ML, zero-shot LLMs/VLMs, and fin... | 187 |
| [llms-as-high-dimensional-nonlinear](skills/llms-as-high-dimensional-nonlinear/SKILL.md) | Analyze, debug, and design LLM systems using the mathematical framework of high-dimensional nonlinear autoregressive mod... | 190 |
| [mas-prove-understanding-process-verification](skills/mas-prove-understanding-process-verification/SKILL.md) | Design and implement process verification for multi-agent LLM systems. Add intermediate-step evaluation to multi-agent w... | 237 |
| [medmo-grounding-understanding-multimodal](skills/medmo-grounding-understanding-multimodal/SKILL.md) | Build medical image analysis pipelines with multi-stage grounded reasoning: cross-modal alignment, instruction-tuned VQA... | 313 |
| [memadapter-fast-alignment-across](skills/memadapter-fast-alignment-across/SKILL.md) | Unify heterogeneous agent memory systems (explicit graphs, parametric weights, latent KV-caches) via generative subgraph... | 166 |
| [merlin-discovery-engine-photonic](skills/merlin-discovery-engine-photonic/SKILL.md) | Build and benchmark photonic quantum machine learning models using the MerLin framework. Integrates linear optical circu... | 248 |
| [mixing-expert-knowledge-bring](skills/mixing-expert-knowledge-bring/SKILL.md) | Integrate domain expert knowledge into LLM fine-tuning pipelines using mixed cold-start SFT and reinforcement learning. ... | 199 |
| [modality-gap-driven-subspace-alignment](skills/modality-gap-driven-subspace-alignment/SKILL.md) | Align multimodal embeddings (vision-language) by correcting the modality gap using the ReAlign/ReVision technique. Fixes... | 231 |
| [muco-multi-turn-contrastive-learning](skills/muco-multi-turn-contrastive-learning/SKILL.md) | Implement multi-turn contrastive learning for multimodal embedding models. Restructures query-target pairs as multi-turn... | 260 |
| [multi-task-grpo-reliable-reasoning](skills/multi-task-grpo-reliable-reasoning/SKILL.md) | | | 228 |
| [multimodal-fine-tuning-synthetic-captions](skills/multimodal-fine-tuning-synthetic-captions/SKILL.md) | Generate synthetic image captions with MLLMs and fine-tune CLIP models using multimodal objectives with supervised contr... | 201 |
| [multimodal-learning-arcing-detection](skills/multimodal-learning-arcing-detection/SKILL.md) | Build multimodal anomaly detection systems that fuse image and sensor data using the MultiDeepSAD framework — a semi-sup... | 215 |
| [omni-rrm-advancing-omni-reward](skills/omni-rrm-advancing-omni-reward/SKILL.md) | Build rubric-grounded reward models and preference evaluation pipelines for multimodal AI outputs. Use when asked to 'ev... | 180 |
| [omnirag-agent-agentic-omnimodal-reasoning](skills/omnirag-agent-agentic-omnimodal-reasoning/SKILL.md) | Build agentic multimodal RAG pipelines that answer questions over long audio-video content under resource constraints. U... | 278 |
| [on-effectiveness-llm-specific-fine-tuning](skills/on-effectiveness-llm-specific-fine-tuning/SKILL.md) | Build and evaluate AI-generated text detectors using LLM-specific fine-tuning strategies. Covers corpus construction, pe... | 171 |
| [one-size-many-fits](skills/one-size-many-fits/SKILL.md) | Build group-aware advertising image generation systems that align diverse user-segment click preferences instead of opti... | 236 |
| [optimizing-small-sample-experience-learning-llm-ba](skills/optimizing-small-sample-experience-learning-llm-ba/SKILL.md) | Implement the ExperienceWeaver hierarchical experience-learning framework to improve text quality from small feedback se... | 195 |
| [out-memory-barrier-highly](skills/out-memory-barrier-highly/SKILL.md) | Configure and run OOMB for memory-efficient long-context LLM training with million-token sequences on limited GPUs. Trig... | 200 |
| [pand-prompt-aware-neighborhood-distillation](skills/pand-prompt-aware-neighborhood-distillation/SKILL.md) | Implement PAND (Prompt-Aware Neighborhood Distillation) for distilling Vision-Language Models into lightweight networks ... | 189 |
| [parameter-efficient-multi-task-fine-tuning-code-re](skills/parameter-efficient-multi-task-fine-tuning-code-re/SKILL.md) | Configure and execute multi-task QLoRA fine-tuning of code models for code generation, translation, and summarization. U... | 223 |
| [parse-open-domain-reasoning-question](skills/parse-open-domain-reasoning-question/SKILL.md) | Build and evaluate reasoning-focused QA systems for low-resource languages using the PARSE methodology: structured promp... | 220 |
| [pathreasoner-r1-instilling-structured-reasoning](skills/pathreasoner-r1-instilling-structured-reasoning/SKILL.md) | Build knowledge-graph-guided structured reasoning pipelines for vision-language models in computational pathology. Imple... | 296 |
| [pcl-reasoner-v15-advancing-math-reasoning-offline](skills/pcl-reasoner-v15-advancing-math-reasoning-offline/SKILL.md) | Implement offline reinforcement learning pipelines for LLM reasoning tasks — decoupling data collection from training fo... | 255 |
| [persodpo-scalable-preference-optimization](skills/persodpo-scalable-preference-optimization/SKILL.md) | Build scalable preference optimization pipelines for persona-grounded dialogue systems using multi-LLM evaluation. Use w... | 173 |
| [persona-driven-data-synthesis-robust-multimodal](skills/persona-driven-data-synthesis-robust-multimodal/SKILL.md) | Generate synthetic training data using controllable persona-driven simulation and Chain-of-Thought reasoning augmentatio... | 161 |
| [phenolip-integrating-phenotype-ontology](skills/phenolip-integrating-phenotype-ontology/SKILL.md) | Build phenotype-aware medical vision-language models by integrating ontology knowledge graphs into CLIP-style pretrainin... | 234 |
| [physprover-advancing-automatic-theorem](skills/physprover-advancing-automatic-theorem/SKILL.md) | Build formal theorem proving pipelines for physics and scientific domains using conjecture-based data generation, Lean 4... | 218 |
| [polarmem-training-free-polarized-latent](skills/polarmem-training-free-polarized-latent/SKILL.md) | Build polarized memory systems for multimodal agents that encode both positive and negative evidence as graph constraint... | 183 |
| [privacy-collapse-benign-fine-tuning](skills/privacy-collapse-benign-fine-tuning/SKILL.md) | Audit fine-tuning datasets and pipelines for privacy collapse — the silent failure where benign training data degrades a... | 195 |
| [promptrl-prompt-matters-rl](skills/promptrl-prompt-matters-rl/SKILL.md) | Implement PromptRL-style joint prompt-refinement + RL training loops for flow-based image generation. Use when the user ... | 196 |
| [protoken-token-level-attribution-federated](skills/protoken-token-level-attribution-federated/SKILL.md) | Implement ProToken-style token-level attribution to trace which federated learning client(s) contributed to each generat... | 164 |
| [pull-requests-as-training](skills/pull-requests-as-training/SKILL.md) | Apply the Clean-PR agentless repo-level code editing protocol: decompose issues into file localization, fine-grained nav... | 199 |
| [r1-syntheticvl-synthetic-data-generative](skills/r1-syntheticvl-synthetic-data-generative/SKILL.md) | Synthesize high-quality multimodal training data using Collective Adversarial Data Synthesis (CADS). Implements a cyclic... | 226 |
| [rapid-real-time-deterministic-trajectory](skills/rapid-real-time-deterministic-trajectory/SKILL.md) | Distill diffusion-based trajectory planners into fast deterministic policies using score-regularized optimization and sa... | 185 |
| [rapo-risk-aware-preference-optimization](skills/rapo-risk-aware-preference-optimization/SKILL.md) | Apply risk-aware preference optimization to make LLM reasoning chains safer against jailbreak attacks. Implements adapti... | 203 |
| [rc-grpo-reward-conditioned-group-relative](skills/rc-grpo-reward-conditioned-group-relative/SKILL.md) | Implement reward-conditioned training pipelines for multi-turn tool-calling agents using RC-GRPO. Injects discrete rewar... | 228 |
| [realistic-synthetic-household-data](skills/realistic-synthetic-household-data/SKILL.md) | Generate realistic synthetic household datasets with bidirectional persona-environment coupling for embodied AI training... | 179 |
| [reasoning-tool-use-compete-agentic](skills/reasoning-tool-use-compete-agentic/SKILL.md) | Diagnose and fix interference between reasoning and tool-use in agentic AI systems using LEAS attribution and DART-style... | 204 |
| [reconstructing-training-data-adapter-based](skills/reconstructing-training-data-adapter-based/SKILL.md) | > | 177 |
| [reinforced-attention-learning](skills/reinforced-attention-learning/SKILL.md) | Implement Reinforced Attention Learning (RAL) for multimodal LLMs — optimize attention distributions via policy gradient... | 216 |
| [reinforcement-learning-backtracking-feedback](skills/reinforcement-learning-backtracking-feedback/SKILL.md) | Implement RLBF (Reinforcement Learning with Backtracking Feedback) for LLM safety — a framework where models learn to de... | 289 |
| [reinforcement-learning-self-distillation](skills/reinforcement-learning-self-distillation/SKILL.md) | Implement Self-Distillation Policy Optimization (SDPO) for RL training loops that convert rich textual feedback into den... | 252 |
| [rethinking-llm-as-a-judge-representation-as-a-judg](skills/rethinking-llm-as-a-judge-representation-as-a-judg/SKILL.md) | Build probing-based evaluation pipelines that judge LLM output quality using hidden-state representations from small lan... | 160 |
| [rethinking-trust-region-reinforcement](skills/rethinking-trust-region-reinforcement/SKILL.md) | Implement Divergence Proximal Policy Optimization (DPPO) for LLM reinforcement learning fine-tuning, replacing PPO's rat... | 204 |
| [revisiting-adaptive-rounding-vectorized](skills/revisiting-adaptive-rounding-vectorized/SKILL.md) | Implement VQRound -- a parameter-efficient adaptive rounding framework for LLM post-training quantization that reparamet... | 175 |
| [reward-designs-general-reasoning](skills/reward-designs-general-reasoning/SKILL.md) | Design and implement likelihood-based reward functions for RL fine-tuning of LLMs on reasoning tasks. Use when: 'design ... | 228 |
| [reward-free-alignment-conflicting-objectives](skills/reward-free-alignment-conflicting-objectives/SKILL.md) | Implement multi-objective LLM alignment using RACO (Reward-free Alignment for Conflicting Objectives) — a method that re... | 239 |
| [reward-inherit-value-biases](skills/reward-inherit-value-biases/SKILL.md) | Audit and mitigate inherited value biases in reward models by analyzing how base-model pretraining shapes RM preferences... | 178 |
| [rewards-as-labels-revisiting](skills/rewards-as-labels-revisiting/SKILL.md) | Implement the REAL (Rewards as Labels) framework for LLM reinforcement learning, which reformulates RLVR policy optimiza... | 218 |
| [risk-awareness-injection-calibrating](skills/risk-awareness-injection-calibrating/SKILL.md) | Implement Risk Awareness Injection (RAI) to defend vision-language models against multimodal jailbreak attacks without r... | 244 |
| [robust-tool-use-fission-grpo](skills/robust-tool-use-fission-grpo/SKILL.md) | | | 320 |
| [scaled-surrogate-gradient-codec-aware-learning](skills/scaled-surrogate-gradient-codec-aware-learning/SKILL.md) | Build end-to-end video processing pipelines that train learned downsamplers/upsamplers through real non-differentiable c... | 215 |
| [sdr-cir-semantic-debias-retrieval](skills/sdr-cir-semantic-debias-retrieval/SKILL.md) | Build training-free composed image retrieval systems that combine a reference image with modification text to find targe... | 171 |
| [se-bench-benchmarking-self-evolution-knowledge](skills/se-bench-benchmarking-self-evolution-knowledge/SKILL.md) | Design knowledge-internalization benchmarks and closed-book training pipelines for LLM self-evolution. Use when: 'build ... | 163 |
| [seccodeprm-process-reward-code](skills/seccodeprm-process-reward-code/SKILL.md) | Step-level security scoring for code generation and vulnerability detection using process reward model techniques. Use w... | 201 |
| [sed-sft-selectively-encouraging-diversity](skills/sed-sft-selectively-encouraging-diversity/SKILL.md) | Implement SED-SFT selective entropy regularization to combat mode collapse in LLM supervised fine-tuning. Use when: 'add... | 229 |
| [self-hinting-enhance-reinforcement-learning](skills/self-hinting-enhance-reinforcement-learning/SKILL.md) | Apply the SAGE self-hinting technique to improve LLM problem-solving by generating graduated hints that boost solution d... | 174 |
| [self-improving-pretraining-post-trained-pretrain](skills/self-improving-pretraining-post-trained-pretrain/SKILL.md) | Build data curation pipelines that use a strong post-trained model to score, filter, and rewrite pretraining corpora for... | 256 |
| [shine-scalable-in-context-hypernetwork](skills/shine-scalable-in-context-hypernetwork/SKILL.md) | Guide Claude to apply SHINE's single-pass context-to-LoRA hypernetwork technique for converting document knowledge into ... | 225 |
| [shopsimulator-evaluating-exploring-rl-driven](skills/shopsimulator-evaluating-exploring-rl-driven/SKILL.md) | Build and evaluate LLM-based shopping assistant agents using structured multi-turn dialogue, personalized product search... | 285 |
| [sicl-at-another-way-adapt](skills/sicl-at-another-way-adapt/SKILL.md) | Adapt auditory LLMs to low-resource speech/audio tasks using Speech In-Context Learning Adaptation Training (SICL-AT). S... | 168 |
| [skillrl-evolving-agents-recursive](skills/skillrl-evolving-agents-recursive/SKILL.md) | Build self-improving agent systems that distill raw execution traces into a hierarchical skill library (SkillBank) and r... | 184 |
| [socratic-geo-synthetic-data-generation](skills/socratic-geo-synthetic-data-generation/SKILL.md) | Generate high-quality synthetic training data through multi-agent feedback loops where a Teacher agent creates parameter... | 226 |
| [spectral-guardrails-agents-wild](skills/spectral-guardrails-agents-wild/SKILL.md) | Implement training-free hallucination detection for LLM agent tool calls using spectral analysis of attention topology. ... | 250 |
| [spell-synthesis-programmatic-edits](skills/spell-synthesis-programmatic-edits/SKILL.md) | Automate library migrations by synthesizing reusable code transformation scripts. Uses LLM-generated migration examples ... | 209 |
| [star-similarity-guided-teacher-assisted-refinement](skills/star-similarity-guided-teacher-assisted-refinement/SKILL.md) | Distill function-calling capabilities from large language models into super-tiny models (0.6B-4B) using the STAR framewo... | 281 |
| [steer2adapt-dynamically-composing-steering](skills/steer2adapt-dynamically-composing-steering/SKILL.md) | Implement the Steer2Adapt framework for adapting LLMs at inference time by dynamically composing steering vectors from a... | 209 |
| [step-35-flash-open](skills/step-35-flash-open/SKILL.md) | Build efficient agentic AI systems using sparse MoE routing, hybrid sliding-window/full attention, multi-token predictio... | 226 |
| [steuerllm-local-specialized-german](skills/steuerllm-local-specialized-german/SKILL.md) | Build domain-specialized LLM pipelines for formal-rule domains (tax law, legal, regulatory) using retrieval-augmented sy... | 203 |
| [syncabel-synthetic-contextualized-augmentation](skills/syncabel-synthetic-contextualized-augmentation/SKILL.md) | Generate synthetic training data for biomedical entity linking using LLM-based contextualized augmentation. Use when: 'g... | 193 |
| [t-llm-teaching-forecast-time](skills/t-llm-teaching-forecast-time/SKILL.md) | Implement temporal distillation pipelines that teach LLMs to forecast time series by training a lightweight trend+freque... | 155 |
| [tamperbench-systematically-stress-testing-safety](skills/tamperbench-systematically-stress-testing-safety/SKILL.md) | Set up and run TamperBench pipelines to systematically stress-test LLM safety under fine-tuning and tampering attacks. U... | 250 |
| [teaching-evaluating-reason-about](skills/teaching-evaluating-reason-about/SKILL.md) | Apply knowledge-augmented reasoning distillation for polymer design tasks. Builds structured Chain-of-Thought pipelines ... | 202 |
| [termigen-high-fidelity-environment-robust](skills/termigen-high-fidelity-environment-robust/SKILL.md) | Synthesize verifiable Docker-based task environments and error-resilient terminal agent trajectories using TermiGen's mu... | 181 |
| [ternarylm-memory-efficient-modeling-native](skills/ternarylm-memory-efficient-modeling-native/SKILL.md) | Implement native 1-bit ternary quantization {-1, 0, +1} for training memory-efficient language models from scratch. Cove... | 230 |
| [the-semantic-trap-fine-tuned](skills/the-semantic-trap-fine-tuned/SKILL.md) | > | 179 |
| [thinking-broad-acting-fast](skills/thinking-broad-acting-fast/SKILL.md) | Build production search relevance systems using Multi-Perspective Chain-of-Thought distillation into lightweight student... | 210 |
| [tracenas-zero-shot-pruning-gradient](skills/tracenas-zero-shot-pruning-gradient/SKILL.md) | Implement TraceNAS-style zero-shot LLM structured pruning using gradient trace correlation as a scale-invariant proxy. J... | 226 |
| [training-data-selection-gradient](skills/training-data-selection-gradient/SKILL.md) | Implement Orthogonal Gradient Selection (OGS) for efficient domain adaptation of LLMs—select training data whose gradien... | 196 |
| [training-multi-turn-search-agent](skills/training-multi-turn-search-agent/SKILL.md) | Build and train multi-turn search agents using BranPO (Branching Relative Policy Optimization) with contrastive dynamic ... | 177 |
| [trapped-past-disentangling-fluid](skills/trapped-past-disentangling-fluid/SKILL.md) | Diagnose whether an LLM is memorizing or reasoning by constructing distributional proximity tests. Classifies task input... | 174 |
| [tre-encouraging-exploration-trust](skills/tre-encouraging-exploration-trust/SKILL.md) | Implement Trust Region Entropy (TRE) for LLM reinforcement learning. Constrains entropy-based exploration to plausible t... | 204 |
| [trifuse-enhancing-attention-based-gui](skills/trifuse-enhancing-attention-based-gui/SKILL.md) | Implement training-free GUI grounding by fusing MLLM attention maps, OCR text cues, and icon caption semantics via Conse... | 162 |
| [ttcs-test-time-curriculum-synthesis](skills/ttcs-test-time-curriculum-synthesis/SKILL.md) | Implement a co-evolving test-time curriculum synthesis framework where a question synthesizer and reasoning solver itera... | 226 |
| [typhoon-s-minimal-open-post-training](skills/typhoon-s-minimal-open-post-training/SKILL.md) | | | 214 |
| [unicomp-unified-evaluation-compression](skills/unicomp-unified-evaluation-compression/SKILL.md) | Guide Claude through evaluating and recommending LLM compression strategies (pruning, quantization, distillation) using ... | 177 |
| [unintended-memorization-sensitive-information](skills/unintended-memorization-sensitive-information/SKILL.md) | Audit fine-tuned LLMs for unintended PII memorization and apply privacy-preserving mitigations. Use when: 'audit my mode... | 199 |
| [v0-generalist-value-any-policy](skills/v0-generalist-value-any-policy/SKILL.md) | Implement V0-style generalist value estimation that profiles any LLM policy from behavioral history rather than paramete... | 169 |
| [vespo-variational-sequence-level-soft](skills/vespo-variational-sequence-level-soft/SKILL.md) | Implement VESPO (Variational Sequence-Level Soft Policy Optimization) for stable off-policy LLM reinforcement learning. ... | 189 |
| [vidvec-unlocking-video-mllm](skills/vidvec-unlocking-video-mllm/SKILL.md) | Extract high-quality video-text embeddings from generative MLLMs using intermediate-layer representations and text-only ... | 197 |
| [visiontrim-unified-vision-token](skills/visiontrim-unified-vision-token/SKILL.md) | Implement VisionTrim's training-free visual token compression for multimodal LLMs. Combines attention-based dominant tok... | 212 |
| [weight-decay-improves-plasticity](skills/weight-decay-improves-plasticity/SKILL.md) | Configure weight decay for optimal model plasticity during LLM pretraining and fine-tuning. Advise on weight decay hyper... | 169 |
| [what-makes-low-bit-quantization-aware](skills/what-makes-low-bit-quantization-aware/SKILL.md) | Implement the Reasoning-QAT pipeline for low-bit quantization-aware training of reasoning LLMs. Combines PTQ initializat... | 189 |
| [when-evaluation-becomes-side](skills/when-evaluation-becomes-side/SKILL.md) | Detect and mitigate regime leakage in AI systems -- the information-theoretic vulnerability where models distinguish eva... | 237 |
| [wildreward-learning-reward-in-the-wild](skills/wildreward-learning-reward-in-the-wild/SKILL.md) | Build reward models from implicit user feedback in chat logs using ordinal regression instead of annotated preference pa... | 175 |
| [winflora-incentivizing-client-adaptive-aggregation](skills/winflora-incentivizing-client-adaptive-aggregation/SKILL.md) | Implement privacy-heterogeneous federated LoRA fine-tuning with noise-aware incentive aggregation (WinFLoRA). Use when: ... | 198 |
| [zero-sum-svd-balancing](skills/zero-sum-svd-balancing/SKILL.md) | Compress LLMs using Zero Sum SVD (ZS-SVD) — a post-training low-rank compression method that globally allocates heteroge... | 236 |
| [zero2text-zero-training-cross-domain-inversion](skills/zero2text-zero-training-cross-domain-inversion/SKILL.md) | Implement embedding inversion attacks that reconstruct original text from vector embeddings without training data, using... | 188 |

---

## Multimodal

**175 skills**

| Skill | Description | Lines |
|-------|-------------|-------|
| [1100-high-efficiency-visual-adapter-complex](skills/1100-high-efficiency-visual-adapter-complex/SKILL.md) | Implement CoLin (Complex Linear Projection) adapters for parameter-efficient fine-tuning of vision foundation models. Ad... | 251 |
| [3d-space-as-scratchpad-editable](skills/3d-space-as-scratchpad-editable/SKILL.md) | Build agentic pipelines that use 3D scene layout as an intermediate reasoning workspace for controllable, spatially-accu... | 172 |
| [a2-llm-end-to-end-conversational-audio-avatar](skills/a2-llm-end-to-end-conversational-audio-avatar/SKILL.md) | Build end-to-end conversational audio avatar systems that jointly generate speech and expressive 3D facial motion from a... | 244 |
| [aegis-governance-integrity-security](skills/aegis-governance-integrity-security/SKILL.md) | Red-team and harden AI voice agents and LLM-powered service systems against adversarial misuse using the Aegis framework... | 237 |
| [agentic-very-long-video](skills/agentic-very-long-video/SKILL.md) | Build agentic systems for understanding very long video streams (hours to weeks) using entity scene graphs, multi-tool p... | 240 |
| [alignment-drift-multimodal-two-phase](skills/alignment-drift-multimodal-two-phase/SKILL.md) | | | 312 |
| [audiorouter-data-audio-understanding](skills/audiorouter-data-audio-understanding/SKILL.md) | Build audio understanding systems that route between internal LLM reasoning and external audio tools using a lightweight... | 249 |
| [audit-after-segmentation-reference-free](skills/audit-after-segmentation-reference-free/SKILL.md) | Build reference-free mask quality assessment pipelines for multimodal segmentation systems. Implements the MQ-Auditor pa... | 251 |
| [autoregressive-yet-revisable-decoding-revision](skills/autoregressive-yet-revisable-decoding-revision/SKILL.md) | | | 201 |
| [avenir-web-human-experience-imitating-multimodal-w](skills/avenir-web-human-experience-imitating-multimodal-w/SKILL.md) | Build robust web automation agents using Mixture of Grounding Experts, experience-imitation planning, and task-tracking ... | 375 |
| [avere-improving-audiovisual-emotion](skills/avere-improving-audiovisual-emotion/SKILL.md) | Build emotion-aware multimodal AI systems that resist spurious cue-emotion associations and hallucinated audiovisual evi... | 177 |
| [bass-benchmarking-audio-lms](skills/bass-benchmarking-audio-lms/SKILL.md) | Build evaluation benchmarks for audio language models using the BASS methodology — structured task taxonomies across str... | 260 |
| [beyond-superficial-unlearning-sharpness-aware](skills/beyond-superficial-unlearning-sharpness-aware/SKILL.md) | Implement sharpness-aware robust erasure (SARE) for hallucination removal in multimodal LLMs. Uses Targeted-SAM to flatt... | 269 |
| [beyond-translation-cross-cultural-meme](skills/beyond-translation-cross-cultural-meme/SKILL.md) | Cross-cultural meme transcreation using a three-stage hybrid pipeline (cultural analysis, visual generation, assembly) t... | 170 |
| [bias-ear-listener-assessing](skills/bias-ear-listener-assessing/SKILL.md) | Assess and audit bias in audio/speech language models using the BiasInEar framework. Evaluate multimodal LLM robustness ... | 182 |
| [bridging-lexical-ambiguity-vision](skills/bridging-lexical-ambiguity-vision/SKILL.md) | Build Visual Word Sense Disambiguation (VWSD) systems that resolve lexical ambiguity using CLIP, diffusion models, and L... | 188 |
| [bridging-modality-gap-roadside](skills/bridging-modality-gap-roadside/SKILL.md) | Build training-free pipelines that convert sparse 3D LiDAR point clouds into depth-encoded 2D images for classification ... | 211 |
| [calliope-tts-based-narrated-e-book](skills/calliope-tts-based-narrated-e-book/SKILL.md) | Build offline TTS-narrated e-books with exact audio-text synchronization in EPUB 3 Media Overlay format. Use when the us... | 178 |
| [can-vision-language-handle-long-context](skills/can-vision-language-handle-long-context/SKILL.md) | Apply visual code compression (LongCodeOCR) to handle long-context code analysis with Vision-Language Models. Renders so... | 185 |
| [chatting-images-introspective-visual](skills/chatting-images-introspective-visual/SKILL.md) | Apply introspective visual thinking by iteratively 'chatting with images' — using language-guided re-examination of visu... | 186 |
| [code2world-gui-world-renderable](skills/code2world-gui-world-renderable/SKILL.md) | Predict and simulate GUI state transitions by generating renderable HTML/CSS/SVG code from screenshots and user actions.... | 292 |
| [codeocr-effectiveness-vision-code](skills/codeocr-effectiveness-vision-code/SKILL.md) | Render source code as images for vision LLM processing to reduce token cost while preserving understanding. Use when: 'r... | 241 |
| [computational-approach-visual-metonymy](skills/computational-approach-visual-metonymy/SKILL.md) | Generate and evaluate visual metonymy -- indirect visual representations that evoke concepts through associated cues rat... | 179 |
| [cord-bridging-audio-text-reasoning](skills/cord-bridging-audio-text-reasoning/SKILL.md) | Implement CORD (Cross-modal On-policy Distillation) to bridge modality gaps in multimodal AI systems. Applies weighted s... | 238 |
| [cure-curriculum-guided-multi-task-training](skills/cure-curriculum-guided-multi-task-training/SKILL.md) | Implement error-aware curriculum learning for multi-task training pipelines, especially medical/vision-language models. ... | 234 |
| [d-orca-dialogue-centric-optimization-robust](skills/d-orca-dialogue-centric-optimization-robust/SKILL.md) | Build dialogue-centric audio-visual captioning pipelines that identify who spoke what and when in multi-party video conv... | 228 |
| [datacross-unified-benchmark-agent](skills/datacross-unified-benchmark-agent/SKILL.md) | Cross-modal data analysis agent that unifies structured sources (SQL, CSV, JSON) with unstructured visual documents (sca... | 203 |
| [decoupling-skeleton-flesh-multimodal](skills/decoupling-skeleton-flesh-multimodal/SKILL.md) | Disentangled structure-content reasoning for table images and structured data. Separates table skeleton (layout/structur... | 180 |
| [deepasmr-llm-based-zero-shot-asmr](skills/deepasmr-llm-based-zero-shot-asmr/SKILL.md) | Build zero-shot ASMR speech generation systems using a two-stage LLM + flow-matching pipeline that separates speaking st... | 227 |
| [deepimagesearch-benchmarking-multimodal-agents](skills/deepimagesearch-benchmarking-multimodal-agents/SKILL.md) | Build agentic image retrieval systems that perform multi-step contextual reasoning over visual histories instead of isol... | 198 |
| [diffa-2-practical-diffusion-general](skills/diffa-2-practical-diffusion-general/SKILL.md) | Design and implement diffusion-based large audio language models (LALMs) using the DIFFA-2 architecture — dual-adapter p... | 237 |
| [diffuspeech-silent-thought-spoken](skills/diffuspeech-silent-thought-spoken/SKILL.md) | | | 276 |
| [discoverllm-executing-intents-discovering](skills/discoverllm-executing-intents-discovering/SKILL.md) | Help users discover and form their intents through adaptive diverge-converge interaction, rather than just asking clarif... | 227 |
| [do-vlms-have-moral](skills/do-vlms-have-moral/SKILL.md) | Audit and harden the moral robustness of Vision-Language Model (VLM) pipelines against adversarial perturbations that fl... | 211 |
| [duogen-general-purpose-interleaved](skills/duogen-general-purpose-interleaved/SKILL.md) | Design and implement interleaved multimodal generation pipelines that alternate between text and image generation using ... | 209 |
| [e2pl-prompt-learning-incomplete](skills/e2pl-prompt-learning-incomplete/SKILL.md) | Design prompt-learning systems for incremental multi-view multi-label classification with missing data. Use when: 'handl... | 188 |
| [edge-optimized-vision-language-underground-infrast](skills/edge-optimized-vision-language-underground-infrast/SKILL.md) | Build edge-deployable two-stage pipelines that combine lightweight segmentation with quantized Vision-Language Models fo... | 483 |
| [emoara-emotion-preserving-english-speech](skills/emoara-emotion-preserving-english-speech/SKILL.md) | Build emotion-preserving cross-lingual speech pipelines that detect emotion from audio, transcribe, translate, and synth... | 217 |
| [emoshift-lightweight-activation-steering](skills/emoshift-lightweight-activation-steering/SKILL.md) | Implement lightweight activation steering for emotion-controllable speech synthesis. Adds learned steering vectors to LL... | 227 |
| [emotion-llamav2-mmeverse-framework-benchmark](skills/emotion-llamav2-mmeverse-framework-benchmark/SKILL.md) | Build multimodal emotion understanding systems using the Emotion-LLaMAv2 architecture and MMEVerse benchmark methodology... | 232 |
| [emotionthinker-prosody-aware-reinforcement-learnin](skills/emotionthinker-prosody-aware-reinforcement-learnin/SKILL.md) | Build prosody-aware speech emotion reasoning pipelines using Chain-of-Thought RL. Implements EmotionThinker's GRPO-PTR t... | 292 |
| [event-vstream-event-driven-real-time-understanding](skills/event-vstream-event-driven-real-time-understanding/SKILL.md) | Build event-driven video stream processing pipelines that detect meaningful state transitions instead of processing ever... | 254 |
| [evocodebench-human-performance-benchmark-self-evol](skills/evocodebench-human-performance-benchmark-self-evol/SKILL.md) | Self-evolving code generation with iterative reflection and revision. Applies a feedback-driven loop where code is submi... | 174 |
| [ex-omni-enabling-3d-facial](skills/ex-omni-enabling-3d-facial/SKILL.md) | Build pipelines that generate synchronized 3D facial animation alongside speech from omni-modal LLMs, using decoupled se... | 204 |
| [explainable-deepfake-detection-rl](skills/explainable-deepfake-detection-rl/SKILL.md) | Build explainable deepfake detection systems using RL-enhanced Self-Blended Images and Chain-of-Thought reasoning. Use w... | 296 |
| [fast-slow-training-multimodal-visual](skills/fast-slow-training-multimodal-visual/SKILL.md) | Implement DualSpeed fast-slow training for multimodal LLMs with visual token pruning. Use when: 'speed up MLLM training'... | 230 |
| [flashvid-video-training-free-tree-based](skills/flashvid-video-training-free-tree-based/SKILL.md) | Accelerate Video Large Language Models (VLLMs) by compressing visual tokens using FlashVID's training-free spatiotempora... | 168 |
| [forest-chat-adapting-vision-language-agents](skills/forest-chat-adapting-vision-language-agents/SKILL.md) | Build LLM-orchestrated agents for bi-temporal satellite image change analysis, combining vision-language models with too... | 362 |
| [found-rl-foundation-model-enhanced-reinforcement](skills/found-rl-foundation-model-enhanced-reinforcement/SKILL.md) | Architect asynchronous VLM-enhanced RL training pipelines that decouple heavy foundation model inference from simulation... | 264 |
| [from-consistency-complementarity-aligned](skills/from-consistency-complementarity-aligned/SKILL.md) | Build multi-modal time series analysis pipelines that align numerical data with visual plots and textual captions using ... | 230 |
| [gamedevbench-evaluating-agentic-capabilities](skills/gamedevbench-evaluating-agentic-capabilities/SKILL.md) | Agentic game development with visual feedback loops for Godot Engine projects. Applies the GameDevBench methodology: nav... | 187 |
| [generative-visual-code-mobile](skills/generative-visual-code-mobile/SKILL.md) | Predict and generate mobile GUI next-states as renderable HTML/CSS code instead of raw pixels. Use when users ask to 'bu... | 297 |
| [genius-generative-fluid-intelligence](skills/genius-generative-fluid-intelligence/SKILL.md) | Evaluate and improve generative AI outputs for fluid intelligence tasks -- pattern induction from context, ad-hoc constr... | 247 |
| [gutenocr-grounded-vision-language-front-end](skills/gutenocr-grounded-vision-language-front-end/SKILL.md) | Build grounded OCR pipelines using GutenOCR's prompt-based interface for reading, detection, and spatial grounding on do... | 210 |
| [harmoni-multimodal-personalization-multi-user](skills/harmoni-multimodal-personalization-multi-user/SKILL.md) | Build multi-user personalization pipelines with per-user profile tracking, multimodal perception, and LLM-driven context... | 197 |
| [history-guided-iterative-visual-reasoning](skills/history-guided-iterative-visual-reasoning/SKILL.md) | | | 170 |
| [ic-eo-interpretable-code-based-assistant](skills/ic-eo-interpretable-code-based-assistant/SKILL.md) | Build conversational Earth Observation agents that turn natural-language queries into executable, auditable Python workf... | 215 |
| [innovator-vl-multimodal-scientific-discovery](skills/innovator-vl-multimodal-scientific-discovery/SKILL.md) | Build data-efficient multimodal scientific ML pipelines using Innovator-VL's principled training methodology. Applies tr... | 247 |
| [instructtime-time-series-classification-multimodal](skills/instructtime-time-series-classification-multimodal/SKILL.md) | Reformulate time series classification as a multimodal generative task using LLMs. Discretizes time series into tokens, ... | 165 |
| [integrating-fine-grained-audio-visual-evidence](skills/integrating-fine-grained-audio-visual-evidence/SKILL.md) | > | 284 |
| [interpreting-controlling-behavior-constitutions](skills/interpreting-controlling-behavior-constitutions/SKILL.md) | Learn and apply natural-language constitutions that map prompt edits to predictable model behavior changes. Use atomic c... | 182 |
| [iterative-refinement-improves-compositional](skills/iterative-refinement-improves-compositional/SKILL.md) | Implement iterative critic-guided refinement loops for compositional image generation. Uses a VLM critic to progressivel... | 199 |
| [jailbreaks-vision-multimodal-reasoning](skills/jailbreaks-vision-multimodal-reasoning/SKILL.md) | > | 250 |
| [kid-knowledge-injected-dual-head-learning](skills/kid-knowledge-injected-dual-head-learning/SKILL.md) | Build knowledge-grounded multimodal content classifiers using the KID dual-head architecture: entity-anchored knowledge ... | 185 |
| [learning-decode-against-compositional](skills/learning-decode-against-compositional/SKILL.md) | Detect and mitigate compositional hallucinations in video multimodal LLM outputs using triple-pathway contrastive decodi... | 284 |
| [less-noise-more-voice](skills/less-noise-more-voice/SKILL.md) | Identify and remove interference tokens from prompts to improve LLM reasoning accuracy. Based on the LENS framework (Les... | 236 |
| [leveraging-data-say-no](skills/leveraging-data-say-no/SKILL.md) | Implement memory-augmented selective prediction for vision-language models using retrieval-based confidence scoring and ... | 194 |
| [linear-merging-unlocks-simple](skills/linear-merging-unlocks-simple/SKILL.md) | Use linear model merging as a cheap proxy for data mixture optimization (DMO) in multimodal LLM fine-tuning. Instead of ... | 168 |
| [lingua-safetybench-benchmark-safety-evaluation-mul](skills/lingua-safetybench-benchmark-safety-evaluation-mul/SKILL.md) | Evaluate and stress-test multilingual vision-language model safety using the Lingua-SafetyBench methodology. Constructs ... | 160 |
| [livibench-omnimodal-benchmark-interactive](skills/livibench-omnimodal-benchmark-interactive/SKILL.md) | Build omnimodal benchmarks and evaluation pipelines for interactive video understanding (livestreams, real-time comments... | 238 |
| [llm-not-all-you](skills/llm-not-all-you/SKILL.md) | Systematic model selection advisor for classification tasks — chooses between classical ML, zero-shot LLMs/VLMs, and fin... | 187 |
| [lmmrec-llm-driven-motivation-aware-multimodal](skills/lmmrec-llm-driven-motivation-aware-multimodal/SKILL.md) | Build motivation-aware recommendation systems that use LLM chain-of-thought prompting to extract user and item motivatio... | 217 |
| [mad-modality-adaptive-decoding-mitigating](skills/mad-modality-adaptive-decoding-mitigating/SKILL.md) | Implement Modality-Adaptive Decoding (MAD) to suppress cross-modal hallucinations in multimodal LLMs. Uses self-assessme... | 227 |
| [mata-trainable-hierarchical-automaton](skills/mata-trainable-hierarchical-automaton/SKILL.md) | Build multi-agent visual reasoning systems using hierarchical finite-state automata with a trainable hyper agent that or... | 303 |
| [medmo-grounding-understanding-multimodal](skills/medmo-grounding-understanding-multimodal/SKILL.md) | Build medical image analysis pipelines with multi-stage grounded reasoning: cross-modal alignment, instruction-tuned VQA... | 313 |
| [medspeak-knowledge-graph-aided-asr](skills/medspeak-knowledge-graph-aided-asr/SKILL.md) | Build knowledge-graph-aided ASR error correction pipelines for medical speech, using phonetic similarity + semantic retr... | 262 |
| [menaspeechbank-reference-voice-bank](skills/menaspeechbank-reference-voice-bank/SKILL.md) | | | 177 |
| [metaphorstar-image-metaphor-understanding](skills/metaphorstar-image-metaphor-understanding/SKILL.md) | Analyze and interpret metaphorical, symbolic, and implied meaning in images using the MetaphorStar visual reasoning chai... | 197 |
| [mmr-bench-comprehensive-benchmark-multimodal](skills/mmr-bench-comprehensive-benchmark-multimodal/SKILL.md) | Build cost-aware multimodal LLM routing systems that select the best model per query based on input signals, budget cons... | 175 |
| [modality-gap-driven-subspace-alignment](skills/modality-gap-driven-subspace-alignment/SKILL.md) | Align multimodal embeddings (vision-language) by correcting the modality gap using the ReAlign/ReVision technique. Fixes... | 231 |
| [muco-multi-turn-contrastive-learning](skills/muco-multi-turn-contrastive-learning/SKILL.md) | Implement multi-turn contrastive learning for multimodal embedding models. Restructures query-target pairs as multi-turn... | 260 |
| [multimodal-fine-tuning-synthetic-captions](skills/multimodal-fine-tuning-synthetic-captions/SKILL.md) | Generate synthetic image captions with MLLMs and fine-tune CLIP models using multimodal objectives with supervised contr... | 201 |
| [multimodal-learning-arcing-detection](skills/multimodal-learning-arcing-detection/SKILL.md) | Build multimodal anomaly detection systems that fuse image and sensor data using the MultiDeepSAD framework — a semi-sup... | 215 |
| [multimodal-multi-agent-ransomware-analysis](skills/multimodal-multi-agent-ransomware-analysis/SKILL.md) | Build multimodal multi-agent pipelines for ransomware classification using specialized per-modality agents, autoencoder ... | 261 |
| [multivis-agent-multi-agent-framework-logic](skills/multivis-agent-multi-agent-framework-logic/SKILL.md) | Build reliable multi-agent data visualization pipelines with logic rule constraints. Use when: 'generate a chart from th... | 193 |
| [now-you-hear-me](skills/now-you-hear-me/SKILL.md) | Audit and defend large audio-language models (LALMs) against narrative-style audio jailbreaks. Based on the 'Now You Hea... | 311 |
| [nwa-mending-spatial-integrity-torn](skills/nwa-mending-spatial-integrity-torn/SKILL.md) | Implement spatially-aware vision token pruning for VLMs using the Nüwa two-stage framework: separation, alignment, and a... | 234 |
| [omni-rrm-advancing-omni-reward](skills/omni-rrm-advancing-omni-reward/SKILL.md) | Build rubric-grounded reward models and preference evaluation pipelines for multimodal AI outputs. Use when asked to 'ev... | 180 |
| [omni-safety-under-cross-modality-conflict](skills/omni-safety-under-cross-modality-conflict/SKILL.md) | Audit and harden omni-modal LLM safety against cross-modality attacks using refusal-vector analysis and OmniSteer alignm... | 217 |
| [omnirag-agent-agentic-omnimodal-reasoning](skills/omnirag-agent-agentic-omnimodal-reasoning/SKILL.md) | Build agentic multimodal RAG pipelines that answer questions over long audio-video content under resource constraints. U... | 278 |
| [one-size-many-fits](skills/one-size-many-fits/SKILL.md) | Build group-aware advertising image generation systems that align diverse user-segment click preferences instead of opti... | 236 |
| [optimizing-small-sample-experience-learning-llm-ba](skills/optimizing-small-sample-experience-learning-llm-ba/SKILL.md) | Implement the ExperienceWeaver hierarchical experience-learning framework to improve text quality from small feedback se... | 195 |
| [pand-prompt-aware-neighborhood-distillation](skills/pand-prompt-aware-neighborhood-distillation/SKILL.md) | Implement PAND (Prompt-Aware Neighborhood Distillation) for distilling Vision-Language Models into lightweight networks ... | 189 |
| [pathreasoner-r1-instilling-structured-reasoning](skills/pathreasoner-r1-instilling-structured-reasoning/SKILL.md) | Build knowledge-graph-guided structured reasoning pipelines for vision-language models in computational pathology. Imple... | 296 |
| [perfguard-performance-aware-agent-visual](skills/perfguard-performance-aware-agent-visual/SKILL.md) | > | 227 |
| [persona-driven-data-synthesis-robust-multimodal](skills/persona-driven-data-synthesis-robust-multimodal/SKILL.md) | Generate synthetic training data using controllable persona-driven simulation and Chain-of-Thought reasoning augmentatio... | 161 |
| [phenolip-integrating-phenotype-ontology](skills/phenolip-integrating-phenotype-ontology/SKILL.md) | Build phenotype-aware medical vision-language models by integrating ontology knowledge graphs into CLIP-style pretrainin... | 234 |
| [phostream-benchmarking-real-world-streaming](skills/phostream-benchmarking-real-world-streaming/SKILL.md) | Build streaming multimodal benchmarks and evaluate omnimodal assistants on continuous audio-visual input with temporal r... | 238 |
| [polarmem-training-free-polarized-latent](skills/polarmem-training-free-polarized-latent/SKILL.md) | Build polarized memory systems for multimodal agents that encode both positive and negative evidence as graph constraint... | 183 |
| [prism-xr-empowering-privacy-aware-xr](skills/prism-xr-empowering-privacy-aware-xr/SKILL.md) | Build privacy-aware pipelines that filter sensitive content from visual frames before sending to cloud AI models, using ... | 301 |
| [promptrl-prompt-matters-rl](skills/promptrl-prompt-matters-rl/SKILL.md) | Implement PromptRL-style joint prompt-refinement + RL training loops for flow-based image generation. Use when the user ... | 196 |
| [r1-syntheticvl-synthetic-data-generative](skills/r1-syntheticvl-synthetic-data-generative/SKILL.md) | Synthesize high-quality multimodal training data using Collective Adversarial Data Synthesis (CADS). Implements a cyclic... | 226 |
| [raicl-retrieval-augmented-in-context-learning](skills/raicl-retrieval-augmented-in-context-learning/SKILL.md) | Build retrieval-augmented in-context learning (RAICL) pipelines that convert time-series or signal data into images and ... | 228 |
| [rapid-real-time-deterministic-trajectory](skills/rapid-real-time-deterministic-trajectory/SKILL.md) | Distill diffusion-based trajectory planners into fast deterministic policies using score-regularized optimization and sa... | 185 |
| [realhd-high-quality-dataset-robust](skills/realhd-high-quality-dataset-robust/SKILL.md) | Detect AI-generated images using NLM noise entropy analysis and build robust forensic detection pipelines. Use when: 'de... | 230 |
| [reasoning-augmented-representations-multimodal-ret](skills/reasoning-augmented-representations-multimodal-ret/SKILL.md) | Decouple reasoning from embedding compression in multimodal retrieval pipelines by enriching queries and corpus entries ... | 223 |
| [reasoning-beyond-literal-cross-style](skills/reasoning-beyond-literal-cross-style/SKILL.md) | Detect and interpret figurative language (sarcasm, humor, offense, metaphor) in multimodal image-text content using a st... | 176 |
| [recgoat-graph-optimal-adaptive](skills/recgoat-graph-optimal-adaptive/SKILL.md) | Build multimodal recommendation systems that align LLM semantic embeddings with collaborative filtering ID features usin... | 182 |
| [regular-variational-latent-reasoning](skills/regular-variational-latent-reasoning/SKILL.md) | Compress verbose chain-of-thought reasoning into compact latent state representations guided by rendered visual summarie... | 236 |
| [reinforced-attention-learning](skills/reinforced-attention-learning/SKILL.md) | Implement Reinforced Attention Learning (RAL) for multimodal LLMs — optimize attention distributions via policy gradient... | 216 |
| [remedit-diffusion-editing-riemannian](skills/remedit-diffusion-editing-riemannian/SKILL.md) | Implement Riemannian-geometry-based diffusion image editing pipelines using geodesic latent navigation, dual-SLERP blend... | 244 |
| [resagent-entropy-based-prior-point](skills/resagent-entropy-based-prior-point/SKILL.md) | Implement entropy-guided coarse-to-fine visual grounding pipelines for referring expression segmentation and point-promp... | 266 |
| [rethinking-genomic-modeling-optical](skills/rethinking-genomic-modeling-optical/SKILL.md) | Implement OpticalDNA-style pipelines that render DNA sequences as 2D visual layouts and process them with OCR-capable vi... | 248 |
| [revisiting-salient-object-detection](skills/revisiting-salient-object-detection/SKILL.md) | Build observer-centric salient object detection systems using the Perceive-Reflect-Adjust agentic loop. Combines a Visio... | 257 |
| [risk-awareness-injection-calibrating](skills/risk-awareness-injection-calibrating/SKILL.md) | Implement Risk Awareness Injection (RAI) to defend vision-language models against multimodal jailbreak attacks without r... | 244 |
| [rvcbench-benchmarking-robustness-voice](skills/rvcbench-benchmarking-robustness-voice/SKILL.md) | Benchmark and harden voice cloning systems against real-world degradation using the RVCBench framework. Evaluates VC mod... | 164 |
| [sar-rag-atr-visual-question](skills/sar-rag-atr-visual-question/SKILL.md) | | | 378 |
| [scaled-surrogate-gradient-codec-aware-learning](skills/scaled-surrogate-gradient-codec-aware-learning/SKILL.md) | Build end-to-end video processing pipelines that train learned downsamplers/upsamplers through real non-differentiable c... | 215 |
| [scratcheval-multimodal-evaluation-framework](skills/scratcheval-multimodal-evaluation-framework/SKILL.md) | Evaluate, debug, and repair block-based Scratch programs using a three-layer executable protocol (VM execution, block-le... | 156 |
| [sdr-cir-semantic-debias-retrieval](skills/sdr-cir-semantic-debias-retrieval/SKILL.md) | Build training-free composed image retrieval systems that combine a reference image with modification text to find targe... | 171 |
| [shotfinder-imagination-driven-open-domain-video](skills/shotfinder-imagination-driven-open-domain-video/SKILL.md) | > | 189 |
| [sicl-at-another-way-adapt](skills/sicl-at-another-way-adapt/SKILL.md) | Adapt auditory LLMs to low-resource speech/audio tasks using Speech In-Context Learning Adaptation Training (SICL-AT). S... | 168 |
| [sonic-o1-real-world-benchmark-evaluating](skills/sonic-o1-real-world-benchmark-evaluating/SKILL.md) | Evaluate multimodal LLMs on audio-video understanding using the SONIC-O1 benchmark framework. Covers three task types: v... | 238 |
| [soundbreak-systematic-study-audio-only](skills/soundbreak-systematic-study-audio-only/SKILL.md) | | | 192 |
| [spatialab-vision-language-perform-spatial](skills/spatialab-vision-language-perform-spatial/SKILL.md) | > | 242 |
| [spava-accelerating-long-video-understanding](skills/spava-accelerating-long-video-understanding/SKILL.md) | Implement Spava-style sequence-parallel approximate attention for accelerating long-video inference across multiple GPUs... | 200 |
| [spd-faith-bench-diagnosing-improving](skills/spd-faith-bench-diagnosing-improving/SKILL.md) | Diagnose and improve faithfulness of chain-of-thought reasoning in multimodal LLM pipelines using the SPD-Faith Bench me... | 237 |
| [spotagent-grounding-visual-geo-localization](skills/spotagent-grounding-visual-geo-localization/SKILL.md) | Build agentic geo-localization systems that combine vision-language model reasoning with tool-assisted verification usin... | 248 |
| [state-art-llm-enabled-interaction](skills/state-art-llm-enabled-interaction/SKILL.md) | Build LLM-powered natural language interfaces for data visualization — NL2VIS pipelines, conversational chart analytics,... | 258 |
| [svrepair-structured-visual-reasoning](skills/svrepair-structured-visual-reasoning/SKILL.md) | Fix bugs using structured visual reasoning -- converts screenshots, control-flow graphs, and UI artifacts into semantic ... | 193 |
| [t2vtree-user-centered-visual-analytics](skills/t2vtree-user-centered-visual-analytics/SKILL.md) | > | 268 |
| [tangrampuzzle-evaluating-multimodal-compositional](skills/tangrampuzzle-evaluating-multimodal-compositional/SKILL.md) | Evaluate and build compositional spatial reasoning systems using geometry-grounded benchmarks and symbolic coordinate fr... | 233 |
| [the-clef-2026-finmmeval-lab](skills/the-clef-2026-finmmeval-lab/SKILL.md) | Build multilingual, multimodal financial AI evaluation pipelines using the FinMMEval framework. Covers financial exam QA... | 246 |
| [thinking-frames-visual-context](skills/thinking-frames-visual-context/SKILL.md) | Decompose complex visual reasoning and spatial planning tasks into frame-by-frame intermediate steps, using visual conte... | 238 |
| [timbre-aware-llm-based-direct-speech-to-speech](skills/timbre-aware-llm-based-direct-speech-to-speech/SKILL.md) | Build direct speech-to-speech translation systems that preserve speaker identity using LLM-based architectures with timb... | 210 |
| [timeblind-spatio-temporal-compositionality-benchma](skills/timeblind-spatio-temporal-compositionality-benchma/SKILL.md) | Build and evaluate spatio-temporal reasoning benchmarks for video LLMs using the TimeBlind minimal-pairs methodology. Ge... | 233 |
| [toward-cognitive-supersensing-multimodal](skills/toward-cognitive-supersensing-multimodal/SKILL.md) | Apply Cognitive Supersensing to multimodal reasoning tasks -- augmenting text-only chain-of-thought with latent visual r... | 173 |
| [towards-understanding-best-practices](skills/towards-understanding-best-practices/SKILL.md) | Quantize vision-language models (VLMs) component-by-component using optimal bit-width strategies derived from multimodal... | 189 |
| [training-multi-turn-search-agent](skills/training-multi-turn-search-agent/SKILL.md) | Build and train multi-turn search agents using BranPO (Branching Relative Policy Optimization) with contrastive dynamic ... | 177 |
| [trifuse-enhancing-attention-based-gui](skills/trifuse-enhancing-attention-based-gui/SKILL.md) | Implement training-free GUI grounding by fusing MLLM attention maps, OCR text cues, and icon caption semantics via Conse... | 162 |
| [ts-debate-multimodal-collaborative-debate](skills/ts-debate-multimodal-collaborative-debate/SKILL.md) | Zero-shot time series reasoning via modality-specialized multi-agent debate. Assigns dedicated text, visual, and numeric... | 232 |
| [tsrbench-comprehensive-multi-task-multi-modal](skills/tsrbench-comprehensive-multi-task-multi-modal/SKILL.md) | Evaluate and build multi-modal time series reasoning pipelines using the TSRBench framework. Covers perception, reasonin... | 206 |
| [ui-venus-15-technical-report](skills/ui-venus-15-technical-report/SKILL.md) | Build GUI automation agents using UI-Venus-1.5 patterns: screenshot-only perception, coordinate-based grounding, traject... | 252 |
| [unikie-bench-benchmarking-multimodal-key](skills/unikie-bench-benchmarking-multimodal-key/SKILL.md) | Extract structured key information from document images using schema-guided prompting for LMMs. Builds KIE pipelines tha... | 292 |
| [unit-based-agent-semi-cascaded-full-duplex](skills/unit-based-agent-semi-cascaded-full-duplex/SKILL.md) | Build full-duplex voice dialogue systems using unit-based agent decomposition and semi-cascaded pipelines. Trigger phras... | 247 |
| [universal-anti-forensics-attack-against](skills/universal-anti-forensics-attack-against/SKILL.md) | Implement ForgeryEraser-style adversarial attacks against AIGC/deepfake detectors by manipulating CLIP embeddings via mu... | 254 |
| [unveiling-cognitive-compass-theory-of-mind-guided](skills/unveiling-cognitive-compass-theory-of-mind-guided/SKILL.md) | Apply Theory-of-Mind (ToM) guided reasoning chains to multimodal emotion analysis tasks. Decomposes emotional reasoning ... | 197 |
| [vectra-metric-dataset-visual](skills/vectra-metric-dataset-visual/SKILL.md) | Assess visual quality of translated product images using Vectra's 14-dimension scoring framework. Use when: 'evaluate tr... | 305 |
| [veq-modality-adaptive-quantization-moe](skills/veq-modality-adaptive-quantization-moe/SKILL.md) | Apply VEQ modality-adaptive quantization to compress MoE Vision-Language Models with minimal accuracy loss. Implements d... | 199 |
| [vica-multimodal-vision-only-cross-attention](skills/vica-multimodal-vision-only-cross-attention/SKILL.md) | Implement and apply the ViCA (Vision-only Cross-Attention) architecture to reduce visual computation in multimodal LLMs ... | 232 |
| [videostf-stress-testing-output-repetition](skills/videostf-stress-testing-output-repetition/SKILL.md) | Detect and stress-test output repetition in Video Large Language Models using n-gram metrics and temporal perturbations ... | 295 |
| [videothinker-building-agentic-videollms](skills/videothinker-building-agentic-videollms/SKILL.md) | Build agentic video understanding systems with LLM-guided tool reasoning. Implements the VideoThinker pattern: confidenc... | 204 |
| [vidvec-unlocking-video-mllm](skills/vidvec-unlocking-video-mllm/SKILL.md) | Extract high-quality video-text embeddings from generative MLLMs using intermediate-layer representations and text-only ... | 197 |
| [villain-at-averimatec-verifying](skills/villain-at-averimatec-verifying/SKILL.md) | Build multi-agent fact-checking pipelines that verify image-text claims through modality-specific analysis, cross-modal ... | 248 |
| [viola-video-in-context-learning](skills/viola-video-in-context-learning/SKILL.md) | Apply the VIOLA framework for label-efficient in-context learning on video or multimodal data. Uses density-uncertainty-... | 221 |
| [vision-deepresearch-benchmark-rethinking-visual-te](skills/vision-deepresearch-benchmark-rethinking-visual-te/SKILL.md) | Build and evaluate Vision-DeepResearch pipelines that combine cropped visual search with multi-hop textual search for ro... | 216 |
| [vision-deepresearch-incentivizing-deepresearch-cap](skills/vision-deepresearch-incentivizing-deepresearch-cap/SKILL.md) | Multi-turn, multi-entity, multi-scale visual and textual deep research agent for answering complex questions about image... | 180 |
| [vision-representations-artificial-intelligence](skills/vision-representations-artificial-intelligence/SKILL.md) | Build autonomous driving safety systems using vision-language models (VLMs) for hazard detection, trajectory planning, a... | 281 |
| [visiontrim-unified-vision-token](skills/visiontrim-unified-vision-token/SKILL.md) | Implement VisionTrim's training-free visual token compression for multimodal LLMs. Combines attention-based dominant tok... | 212 |
| [visor-visual-spatial-object](skills/visor-visual-spatial-object/SKILL.md) | Implement VISOR-style three-stage visual spatial reasoning (think, think-summary, action) for embodied navigation and ob... | 198 |
| [vistira-closing-image-text-modality](skills/vistira-closing-image-text-modality/SKILL.md) | Solve math problems from images by decomposing them into interleaved natural-language rationales and executable Python c... | 189 |
| [visual-cognitive-demands-model-powered](skills/visual-cognitive-demands-model-powered/SKILL.md) | Evaluate visual and cognitive demands of in-vehicle LLM interfaces using the Monk et al. (2026) dual-metric framework. I... | 290 |
| [visual-reasoning-over-time](skills/visual-reasoning-over-time/SKILL.md) | Analyze time series data using the MAS4TS Analyzer-Reasoner-Executor multi-agent paradigm: convert series to plots, extr... | 180 |
| [vividface-real-time-realistic-facial](skills/vividface-real-time-realistic-facial/SKILL.md) | Build real-time facial expression shadowing pipelines for humanoid robots using the VividFace two-module architecture (m... | 277 |
| [vividvoice-unified-framework-scene-aware](skills/vividvoice-unified-framework-scene-aware/SKILL.md) | Build scene-aware speech synthesis systems that generate speech conditioned on visual scenes, aligning timbre and enviro... | 245 |
| [vlm-guided-iterative-refinement-surgical](skills/vlm-guided-iterative-refinement-surgical/SKILL.md) | Build iterative VLM-guided refinement pipelines for image segmentation tasks, especially surgical/medical imagery. Uses ... | 255 |
| [vln-pilot-vision-language-as-autonomous](skills/vln-pilot-vision-language-as-autonomous/SKILL.md) | Build VLLM-driven autonomous navigation agents that interpret natural language instructions and ground them in visual ob... | 235 |
| [vowelprompt-hearing-speech-emotions](skills/vowelprompt-hearing-speech-emotions/SKILL.md) | Build speech emotion recognition pipelines that augment LLMs with vowel-level prosodic features converted to natural lan... | 287 |
| [voxmorph-scalable-zero-shot-voice](skills/voxmorph-scalable-zero-shot-voice/SKILL.md) | Build and deploy zero-shot voice identity morphing pipelines using disentangled prosody/timbre embeddings and Spherical ... | 203 |
| [vtc-r1-vision-text-compression-long-context](skills/vtc-r1-vision-text-compression-long-context/SKILL.md) | Implement VTC-R1 vision-text compression for efficient long-context reasoning. Renders intermediate reasoning segments i... | 269 |
| [wavlink-compact-audio-text-embeddings](skills/wavlink-compact-audio-text-embeddings/SKILL.md) | Build compact audio-text embedding systems using WavLink's global Whisper token architecture with Matryoshka dimensional... | 197 |
| [when-much-imagine-adaptive](skills/when-much-imagine-adaptive/SKILL.md) | Adaptive test-time scaling framework that decides WHEN and HOW MUCH to invoke expensive generative steps (world models, ... | 228 |
| [xai-clip-roi-guided-perturbation-framework](skills/xai-clip-roi-guided-perturbation-framework/SKILL.md) | Build ROI-guided perturbation pipelines for explainable medical image segmentation using CLIP embeddings. Generates boun... | 226 |
| [xlist-hate-checklist-based-framework-interpretable](skills/xlist-hate-checklist-based-framework-interpretable/SKILL.md) | Decompose hate speech detection into a checklist of ten concept-level binary questions answered independently by an LLM,... | 229 |
| [zero-shot-product-attribute-labeling](skills/zero-shot-product-attribute-labeling/SKILL.md) | Extract and classify product attributes from images using Vision-Language Models with structured prompts and a three-tie... | 268 |

---

## Multi-Agent Systems

**124 skills**

| Skill | Description | Lines |
|-------|-------------|-------|
| [adaptive-confidence-gating-multi-agent](skills/adaptive-confidence-gating-multi-agent/SKILL.md) | Multi-agent code generation using structured debate with adaptive confidence gating. Three specialized agents (User/Prod... | 184 |
| [adareasoner-dynamic-tool-orchestration](skills/adareasoner-dynamic-tool-orchestration/SKILL.md) | Adaptive multi-step tool orchestration for complex reasoning tasks. Dynamically selects, sequences, and composes tools b... | 168 |
| [agent-primitives-reusable-latent](skills/agent-primitives-reusable-latent/SKILL.md) | Design and orchestrate multi-agent systems using reusable Agent Primitives (Review, Voting/Selection, Planning/Execution... | 252 |
| [agent2agent-threats-safety-critical-assistants](skills/agent2agent-threats-safety-critical-assistants/SKILL.md) | Threat model multi-agent LLM systems using the AgentHeLLM framework -- formally separating asset identification from att... | 207 |
| [agentark-distilling-multi-agent-intelligence](skills/agentark-distilling-multi-agent-intelligence/SKILL.md) | Distill multi-agent debate reasoning into a single LLM's behavior. Apply AgentArk's three-tier distillation strategy to ... | 199 |
| [agentic-ai-healthcare-medicine](skills/agentic-ai-healthcare-medicine/SKILL.md) | Design, evaluate, and improve LLM-based agentic systems for healthcare using a seven-dimensional taxonomy with 29 sub-di... | 274 |
| [agentic-reinforcement-learning-empowers](skills/agentic-reinforcement-learning-empowers/SKILL.md) | Build tool-augmented agent systems that decouple domain reasoning from knowledge storage, following the ChemCRAFT patter... | 242 |
| [agenticpay-multi-agent-negotiation-system](skills/agenticpay-multi-agent-negotiation-system/SKILL.md) | Build multi-agent LLM negotiation systems where buyer and seller agents reach deals through natural language. Use when a... | 226 |
| [agenticsimlaw-juvenile-courtroom-multi-agent](skills/agenticsimlaw-juvenile-courtroom-multi-agent/SKILL.md) | Structured multi-agent courtroom debate for explainable high-stakes tabular decisions. Use when: 'set up a multi-agent d... | 181 |
| [agyn-multi-agent-system-team-based](skills/agyn-multi-agent-system-team-based/SKILL.md) | Orchestrate multi-agent teams for autonomous software engineering using the Agyn methodology: coordinator, researcher, i... | 202 |
| [aidev-studying-ai-coding](skills/aidev-studying-ai-coding/SKILL.md) | Analyze AI coding agent activity on GitHub repositories using the AIDev methodology. Identify agentic PRs, measure agent... | 195 |
| [ama-adaptive-memory-multi-agent](skills/ama-adaptive-memory-multi-agent/SKILL.md) | Build adaptive memory systems using coordinated multi-agent collaboration with hierarchical storage and consistency main... | 224 |
| [aorchestra-automating-sub-agent-creation](skills/aorchestra-automating-sub-agent-creation/SKILL.md) | Dynamically create specialized sub-agents for complex multi-step tasks using the AOrchestra pattern: decompose goals, th... | 203 |
| [automated-multiple-mini-interview](skills/automated-multiple-mini-interview/SKILL.md) | Multi-agent framework for scoring subjective, open-ended responses (interviews, essays, reflections) using transcript re... | 189 |
| [automated-rubrics-reliable-evaluation](skills/automated-rubrics-reliable-evaluation/SKILL.md) | Generate fine-grained evaluation rubrics for medical dialogue systems using a retrieval-augmented multi-agent pipeline. ... | 180 |
| [autonomous-data-processing-meta-agents](skills/autonomous-data-processing-meta-agents/SKILL.md) | Build self-managing data processing pipelines using hierarchical meta-agent orchestration. Decomposes complex data tasks... | 216 |
| [autonomous-multi-agent-ai-high-throughput](skills/autonomous-multi-agent-ai-high-throughput/SKILL.md) | | | 189 |
| [bass-benchmarking-audio-lms](skills/bass-benchmarking-audio-lms/SKILL.md) | Build evaluation benchmarks for audio language models using the BASS methodology — structured task taxonomies across str... | 260 |
| [beyond-accuracy-cognitive-load](skills/beyond-accuracy-cognitive-load/SKILL.md) | Analyze and reduce cognitive load in tool-use agent workflows using the Cognitive Load Framework from AAAI 2026. Diagnos... | 218 |
| [blind-gods-broken-screens](skills/blind-gods-broken-screens/SKILL.md) | Architect secure, intent-centric agent systems using the Aura pattern: Hub-and-Spoke agent topology, cryptographic ident... | 219 |
| [cam-causality-based-analysis-framework](skills/cam-causality-based-analysis-framework/SKILL.md) | Analyze and optimize multi-agent code generation pipelines using causality-based importance ranking of intermediate feat... | 153 |
| [co-redteam-orchestrated-security-discovery](skills/co-redteam-orchestrated-security-discovery/SKILL.md) | Multi-agent security vulnerability discovery and exploitation using Co-RedTeam's orchestrated workflow. Decomposes secur... | 197 |
| [cognitive-platform-engineering-autonomous](skills/cognitive-platform-engineering-autonomous/SKILL.md) | Build autonomous cloud operations using a four-plane cognitive architecture (Sensing, Reasoning, Orchestration, Experien... | 247 |
| [colt-lightweight-multi-llm-collaboration](skills/colt-lightweight-multi-llm-collaboration/SKILL.md) | | | 190 |
| [commcp-multi-agent-coordination-llm-based](skills/commcp-multi-agent-coordination-llm-based/SKILL.md) | Build decentralized multi-agent coordination systems using LLM-based communication calibrated with conformal prediction.... | 225 |
| [completing-missing-annotation-multi-agent](skills/completing-missing-annotation-multi-agent/SKILL.md) | Multi-agent debate framework for relevance assessment and annotation completion. Uses opposing-stance LLM agents with it... | 251 |
| [constrained-process-maps-multi-agent](skills/constrained-process-maps-multi-agent/SKILL.md) | Build multi-agent workflows structured as constrained DAG process maps with Monte Carlo uncertainty estimation. Each age... | 244 |
| [contextevolve-multi-agent-context-compression](skills/contextevolve-multi-agent-context-compression/SKILL.md) | Multi-agent iterative code optimization using context compression. Decomposes optimization into three agents (Summarizer... | 175 |
| [core-ubiquitous-6g-intelligence](skills/core-ubiquitous-6g-intelligence/SKILL.md) | Design and implement multi-LLM agent orchestration systems over hierarchical compute tiers using the CORE framework patt... | 195 |
| [cowork-x-experience-optimized-co-evolution-multi-a](skills/cowork-x-experience-optimized-co-evolution-multi-a/SKILL.md) | Build multi-agent collaboration systems with experience-driven co-evolution using HTN skill libraries and post-episode o... | 150 |
| [darwin-dynamic-agentically-rewriting](skills/darwin-dynamic-agentically-rewriting/SKILL.md) | Evolutionary multi-agent code optimization using genetic algorithms. Agents mutate each other's training/configuration c... | 167 |
| [data-centric-interpretability-llm-based-multi-agen](skills/data-centric-interpretability-llm-based-multi-agen/SKILL.md) | Analyze LLM agent behavior across training runs using Sparse Autoencoder (SAE) features and LLM-summarizer pipelines. Gr... | 189 |
| [dllm-agent-see-farther](skills/dllm-agent-see-farther/SKILL.md) | Design and implement multi-agent workflows using the DeepDiver hierarchical orchestration pattern with diffusion-inspire... | 163 |
| [dr-mas-stable-reinforcement-learning](skills/dr-mas-stable-reinforcement-learning/SKILL.md) | Design and implement stable reinforcement learning pipelines for multi-agent LLM systems using agent-wise advantage norm... | 203 |
| [dynamic-role-assignment-multi-agent](skills/dynamic-role-assignment-multi-agent/SKILL.md) | Dynamically assign specialized roles to multiple AI agents via a meta-debate protocol (proposal + peer review) before ru... | 182 |
| [eft-cot-multi-agent-chain-of-thought-framework](skills/eft-cot-multi-agent-chain-of-thought-framework/SKILL.md) | Build multi-agent emotion-focused therapy (EFT) reasoning pipelines for empathetic mental health Q&A systems. Uses a bot... | 300 |
| [emotion-llamav2-mmeverse-framework-benchmark](skills/emotion-llamav2-mmeverse-framework-benchmark/SKILL.md) | Build multimodal emotion understanding systems using the Emotion-LLaMAv2 architecture and MMEVerse benchmark methodology... | 232 |
| [epistemic-context-learning-building](skills/epistemic-context-learning-building/SKILL.md) | Build trust-aware multi-agent systems using Epistemic Context Learning (ECL). Constructs peer reliability profiles from ... | 210 |
| [evoconfig-self-evolving-multi-agent-systems](skills/evoconfig-self-evolving-multi-agent-systems/SKILL.md) | Autonomous environment configuration using multi-agent diagnosis and self-evolving error repair. Use when: 'set up the d... | 194 |
| [experience-driven-multi-agent-systems-training-fre](skills/experience-driven-multi-agent-systems-training-fre/SKILL.md) | Build self-evolving multi-agent systems that accumulate tool-level expertise through structured interaction without mode... | 168 |
| [farm-field-aware-resolution-intelligent](skills/farm-field-aware-resolution-intelligent/SKILL.md) | Build intelligent trigger-action automation systems using FARM's two-stage architecture: contrastive retrieval + multi-a... | 187 |
| [fat-cat-document-driven-metacognitive-multi-agent](skills/fat-cat-document-driven-metacognitive-multi-agent/SKILL.md) | > | 225 |
| [flyaoc-evaluating-agentic-ontology](skills/flyaoc-evaluating-agentic-ontology/SKILL.md) | Build multi-agent systems for end-to-end ontology curation from scientific literature. Applies FlyAOC's agent architectu... | 184 |
| [forest-chat-adapting-vision-language-agents](skills/forest-chat-adapting-vision-language-agents/SKILL.md) | Build LLM-orchestrated agents for bi-temporal satellite image change analysis, combining vision-language models with too... | 362 |
| [from-assumptions-actions-turning](skills/from-assumptions-actions-turning/SKILL.md) | Build uncertainty-aware planners for multi-agent systems using the PCE (Planner-Composer-Evaluator) decision tree framew... | 242 |
| [from-prompt-response-goal-directed-systems](skills/from-prompt-response-goal-directed-systems/SKILL.md) | Design production-grade agentic AI architectures with separated cognition/execution layers, typed tool interfaces, multi... | 177 |
| [gametalk-training-strategic-conversation](skills/gametalk-training-strategic-conversation/SKILL.md) | Build multi-agent strategic conversation systems where LLMs negotiate, coordinate, and optimize long-term objectives thr... | 207 |
| [graphagents-knowledge-graph-guided-agentic](skills/graphagents-knowledge-graph-guided-agentic/SKILL.md) | Build multi-agent pipelines that use knowledge graphs to guide LLM reasoning across domains. Agents specialize in proble... | 185 |
| [hidden-licensing-risks-llmware](skills/hidden-licensing-risks-llmware/SKILL.md) | Detect license incompatibilities across LLM supply chains (OSS repos, models, datasets) using the LiAgent multi-agent ex... | 182 |
| [inficoevalchain-blockchain-based-decentralized-fra](skills/inficoevalchain-blockchain-based-decentralized-fra/SKILL.md) | Design and implement decentralized LLM evaluation systems using blockchain-based consensus, multi-node validation, and s... | 202 |
| [internalizing-multi-agent-reasoning-accurate](skills/internalizing-multi-agent-reasoning-accurate/SKILL.md) | Distill multi-agent reasoning into a single efficient model for recommendation or retrieval. Use when: 'build a recommen... | 174 |
| [interpreting-agentic-systems-beyond](skills/interpreting-agentic-systems-beyond/SKILL.md) | Audit and instrument agentic AI systems for system-level interpretability and accountability. Embeds traceability, causa... | 329 |
| [learning-compose-cross-domain-agentic](skills/learning-compose-cross-domain-agentic/SKILL.md) | Generate cross-domain agentic workflows using decompose-recompose-decide composition over reusable capability bases. Use... | 159 |
| [learning-decentralized-collaboration-multi-agent](skills/learning-decentralized-collaboration-multi-agent/SKILL.md) | Design and orchestrate decentralized multi-LLM collaboration systems using Multi-Agent Actor-Critic (MAAC) patterns from... | 218 |
| [legalmalr-multi-agent-query-understanding](skills/legalmalr-multi-agent-query-understanding/SKILL.md) | Multi-agent query reformulation and LLM reranking for retrieval over legal, regulatory, or domain-specific corpora. Use ... | 168 |
| [lemon-agent-technical-report](skills/lemon-agent-technical-report/SKILL.md) | Orchestrate multi-agent workflows using the Lemon Agent orchestrator-worker pattern with hierarchical scheduling, progre... | 186 |
| [lingxidiagbench-multi-agent-framework-benchmarking](skills/lingxidiagbench-multi-agent-framework-benchmarking/SKILL.md) | Build multi-agent benchmarking systems with role-separated agents (simulator, interviewer, evaluator) for structured mul... | 216 |
| [livemedbench-contamination-free-medical-benchmark](skills/livemedbench-contamination-free-medical-benchmark/SKILL.md) | Build contamination-free LLM evaluation pipelines with multi-agent data curation and automated rubric-based scoring. Use... | 296 |
| [livibench-omnimodal-benchmark-interactive](skills/livibench-omnimodal-benchmark-interactive/SKILL.md) | Build omnimodal benchmarks and evaluation pipelines for interactive video understanding (livestreams, real-time comments... | 238 |
| [llms-as-orchestrators-constraint-compliant](skills/llms-as-orchestrators-constraint-compliant/SKILL.md) | | | 186 |
| [localv-exploiting-information-locality](skills/localv-exploiting-information-locality/SKILL.md) | Multi-agent framework for generating large-scale Verilog/RTL code from long hardware specifications by decomposing long-... | 138 |
| [longcat-flash-thinking-2601-technical-report](skills/longcat-flash-thinking-2601-technical-report/SKILL.md) | Build robust multi-tool agentic pipelines with noise-aware execution, parallel reasoning, and environment scaling patter... | 311 |
| [marble-multi-agent-reasoning-bioinformatics](skills/marble-multi-agent-reasoning-bioinformatics/SKILL.md) | Iteratively refine bioinformatics and ML models using MARBLE's multi-agent debate framework with role-specialized agents... | 206 |
| [marti-mars2-scaling-multi-agent-self-search-reinfo](skills/marti-mars2-scaling-multi-agent-self-search-reinfo/SKILL.md) | Multi-agent tree-search code generation using heterogeneous agent collaboration with error-feedback refinement. Spawns m... | 204 |
| [mas-prove-understanding-process-verification](skills/mas-prove-understanding-process-verification/SKILL.md) | Design and implement process verification for multi-agent LLM systems. Add intermediate-step evaluation to multi-agent w... | 237 |
| [mascot-multi-agent-socio-collaborative-companion](skills/mascot-multi-agent-socio-collaborative-companion/SKILL.md) | Design and orchestrate multi-agent companion systems where each agent maintains a distinct persona and contributes diver... | 244 |
| [mata-multiagent-framework-for](skills/mata-multiagent-framework-for/SKILL.md) | Multi-agent table question answering using MATA's three-path reasoning strategy (Chain-of-Thought, Program-of-Thought, T... | 167 |
| [mata-trainable-hierarchical-automaton](skills/mata-trainable-hierarchical-automaton/SKILL.md) | Build multi-agent visual reasoning systems using hierarchical finite-state automata with a trainable hyper agent that or... | 303 |
| [mathliblemma-folklore-lemma-generation](skills/mathliblemma-folklore-lemma-generation/SKILL.md) | Multi-agent system for discovering and formalizing missing 'folklore' lemmas in Lean 4 / Mathlib. Identifies gaps in for... | 181 |
| [menvagent-scalable-polyglot-environment](skills/menvagent-scalable-polyglot-environment/SKILL.md) | Automated Docker environment construction for polyglot repositories using a Planning-Execution-Verification multi-agent ... | 188 |
| [mermaid-memory-enhanced-retrieval-reasoning](skills/mermaid-memory-enhanced-retrieval-reasoning/SKILL.md) | Memory-enhanced multi-agent retrieval and reasoning for veracity assessment and fact-checking. Use when: 'verify this cl... | 189 |
| [mirror-multi-agent-framework-iterative](skills/mirror-multi-agent-framework-iterative/SKILL.md) | Translate natural language optimization problems into mathematical models and solver code using MIRROR's multi-agent pip... | 166 |
| [moco-one-stop-shop-collaboration](skills/moco-one-stop-shop-collaboration/SKILL.md) | Design and implement multi-LM collaboration pipelines using the MoCo framework's 26 methods across four collaboration le... | 225 |
| [multi-agent-causal-reasoning-system](skills/multi-agent-causal-reasoning-system/SKILL.md) | Build multi-agent systems that discover causal rules from event sequences using specialized agents (causal discovery, co... | 225 |
| [multi-agent-collaborative-intrusion-detection](skills/multi-agent-collaborative-intrusion-detection/SKILL.md) | Build multi-agent intrusion detection systems using LLM-enhanced collaborative agents for network traffic classification... | 309 |
| [multi-agent-constraint-factorization-reveals](skills/multi-agent-constraint-factorization-reveals/SKILL.md) | Orchestrate multi-agent LLM pipelines using constraint factorization -- decomposing complex requirements into separate c... | 157 |
| [multi-agent-end-to-end-vulnerability-management](skills/multi-agent-end-to-end-vulnerability-management/SKILL.md) | Detect, confirm, repair, and validate recurring software vulnerabilities using a multi-agent pipeline modeled on MAVM. B... | 196 |
| [multi-agent-teams-hold-experts](skills/multi-agent-teams-hold-experts/SKILL.md) | Prevent expertise dilution in multi-agent LLM workflows by applying findings from 'Multi-Agent Teams Hold Experts Back' ... | 151 |
| [multi-agentic-ai-fairness-aware-accelerated](skills/multi-agentic-ai-fairness-aware-accelerated/SKILL.md) | Design and implement multi-agent systems for fairness-aware, low-latency inference orchestration across distributed edge... | 200 |
| [multimodal-multi-agent-ransomware-analysis](skills/multimodal-multi-agent-ransomware-analysis/SKILL.md) | Build multimodal multi-agent pipelines for ransomware classification using specialized per-modality agents, autoencoder ... | 261 |
| [multivis-agent-multi-agent-framework-logic](skills/multivis-agent-multi-agent-framework-logic/SKILL.md) | Build reliable multi-agent data visualization pipelines with logic rule constraints. Use when: 'generate a chart from th... | 193 |
| [mulvul-retrieval-augmented-multi-agent-code](skills/mulvul-retrieval-augmented-multi-agent-code/SKILL.md) | Multi-agent vulnerability detection using coarse-to-fine routing, contrastive retrieval, and cross-model prompt evolutio... | 204 |
| [on-uncertainty-model-based-multi-agent](skills/on-uncertainty-model-based-multi-agent/SKILL.md) | Apply entropy-based uncertainty analysis to multi-agent LLM systems. Diagnose when multi-agent setups hurt performance, ... | 184 |
| [openguandan-large-scale-imperfect-information](skills/openguandan-large-scale-imperfect-information/SKILL.md) | Build AI agents for the OpenGuanDan imperfect-information card game benchmark. Covers WebSocket client implementation, g... | 366 |
| [optimizing-small-sample-experience-learning-llm-ba](skills/optimizing-small-sample-experience-learning-llm-ba/SKILL.md) | Implement the ExperienceWeaver hierarchical experience-learning framework to improve text quality from small feedback se... | 195 |
| [pamas-self-adaptive-multi-agent-system](skills/pamas-self-adaptive-multi-agent-system/SKILL.md) | Build hierarchical multi-agent systems that detect misinformation, anomalies, and deceptive content using perspective-aw... | 162 |
| [paperbanana-automating-academic-illustration](skills/paperbanana-automating-academic-illustration/SKILL.md) | Generate publication-ready academic illustrations using a multi-agent pipeline inspired by PaperBanana. Orchestrates ret... | 197 |
| [pathwise-planning-world-automated](skills/pathwise-planning-world-automated/SKILL.md) | Multi-agent heuristic design framework that uses an entailment graph, policy/world-model/critic agents, and routed refle... | 165 |
| [pearl-plan-exploration-adaptive](skills/pearl-plan-exploration-adaptive/SKILL.md) | Apply PEARL's two-phase tool orchestration: offline tool exploration to learn valid usage patterns and failure modes, th... | 172 |
| [prism-principled-framework-multi-agent](skills/prism-principled-framework-multi-agent/SKILL.md) | > | 138 |
| [prism-xr-empowering-privacy-aware-xr](skills/prism-xr-empowering-privacy-aware-xr/SKILL.md) | Build privacy-aware pipelines that filter sensitive content from visual frames before sending to cloud AI models, using ... | 301 |
| [quasar-universal-autonomous-system](skills/quasar-universal-autonomous-system/SKILL.md) | Build autonomous multi-scale scientific simulation pipelines using the QUASAR architecture: a Strategist-Operator-Evalua... | 165 |
| [rank-and-reason-multi-agent-collaboration-accelera](skills/rank-and-reason-multi-agent-collaboration-accelera/SKILL.md) | | | 245 |
| [refer-agent-collaborative-multi-agent-system](skills/refer-agent-collaborative-multi-agent-system/SKILL.md) | Build collaborative multi-agent systems that use alternating reasoning-reflection cycles with specialized agent roles, c... | 179 |
| [refuge-feature-generation-prediction](skills/refuge-feature-generation-prediction/SKILL.md) | Automated feature engineering for prediction tasks on relational databases using a multi-agent LLM pipeline. Generates, ... | 168 |
| [roma-recursive-open-meta-agent](skills/roma-recursive-open-meta-agent/SKILL.md) | Decompose long-horizon, multi-step tasks using ROMA's recursive meta-agent pattern: Atomizer decides if a task needs spl... | 185 |
| [rulesmith-multi-agent-automated-game](skills/rulesmith-multi-agent-automated-game/SKILL.md) | Automated game balancing using multi-agent LLM self-play coupled with Bayesian optimization. Use when the user asks to '... | 193 |
| [shardmemo-masked-moe-routing](skills/shardmemo-masked-moe-routing/SKILL.md) | Implement ShardMemo-style tiered, sharded memory with masked Mixture-of-Experts routing for agentic LLM systems. Use whe... | 176 |
| [socialveil-probing-social-intelligence](skills/socialveil-probing-social-intelligence/SKILL.md) | Stress-test LLM agents' social intelligence by injecting realistic communication barriers (semantic vagueness, sociocult... | 203 |
| [socratic-geo-synthetic-data-generation](skills/socratic-geo-synthetic-data-generation/SKILL.md) | Generate high-quality synthetic training data through multi-agent feedback loops where a Teacher agent creates parameter... | 226 |
| [solagent-specialized-multi-agent-framework](skills/solagent-specialized-multi-agent-framework/SKILL.md) | Generate secure, functionally correct Solidity smart contracts using a dual-loop refinement process: an inner loop that ... | 193 |
| [sparc-rag-adaptive-sequential-parallel-scaling](skills/sparc-rag-adaptive-sequential-parallel-scaling/SKILL.md) | Implement multi-agent RAG systems with coordinated sequential-parallel scaling and shared context management for complex... | 248 |
| [status-hierarchies](skills/status-hierarchies/SKILL.md) | Detect and mitigate status hierarchy bias in multi-agent LLM systems. Applies expectation states theory to audit deferen... | 229 |
| [supchain-bench-benchmarking-real-world-supply](skills/supchain-bench-benchmarking-real-world-supply/SKILL.md) | Build reliable long-horizon supply chain agents using the SupChain-ReAct pattern: multi-path ReAct trajectories with maj... | 198 |
| [sycoeval-em-sycophancy-evaluation-simulated](skills/sycoeval-em-sycophancy-evaluation-simulated/SKILL.md) | Build multi-agent adversarial simulations to evaluate LLM sycophancy and policy compliance under social pressure. Use wh... | 241 |
| [symphony-synergistic-multi-agent-planning](skills/symphony-synergistic-multi-agent-planning/SKILL.md) | > | 201 |
| [synthagent-multi-agent-framework-realistic](skills/synthagent-multi-agent-framework-realistic/SKILL.md) | Build multi-agent pipelines that generate realistic synthetic patient profiles by integrating epidemiological data, medi... | 298 |
| [table-as-search-formulate-long-horizon-agentic](skills/table-as-search-formulate-long-horizon-agentic/SKILL.md) | Structured table-completion framework for long-horizon information seeking. Converts complex research queries into datab... | 203 |
| [termigen-high-fidelity-environment-robust](skills/termigen-high-fidelity-environment-robust/SKILL.md) | Synthesize verifiable Docker-based task environments and error-resilient terminal agent trajectories using TermiGen's mu... | 181 |
| [thinktank-me-multi-expert-framework-middle](skills/thinktank-me-multi-expert-framework-middle/SKILL.md) | Build multi-expert forecasting systems where specialized LLM agents collaborate through routing and aggregation to predi... | 207 |
| [tokenomics-quantifying-where-tokens](skills/tokenomics-quantifying-where-tokens/SKILL.md) | Analyze and optimize token consumption in LLM-based multi-agent software engineering workflows. Maps agent execution tra... | 227 |
| [toward-culturally-aligned-ontology-guided](skills/toward-culturally-aligned-ontology-guided/SKILL.md) | Ontology-guided multi-agent reasoning for culturally aligned LLM outputs. Use when building systems that must respect cu... | 190 |
| [towards-adaptive-scalable-robust](skills/towards-adaptive-scalable-robust/SKILL.md) | Implement RAPS (Reputation-Aware Publish-Subscribe) multi-agent coordination using intent-based pub/sub messaging, react... | 205 |
| [towards-declarative-agentic-layer](skills/towards-declarative-agentic-layer/SKILL.md) | Build grounded, declarative agentic architectures using the DALIA pattern: capability descriptors, discovery protocols, ... | 218 |
| [tracecoder-trace-driven-multi-agent-framework](skills/tracecoder-trace-driven-multi-agent-framework/SKILL.md) | Trace-driven debugging framework for LLM-generated code. Uses diagnostic probe instrumentation, causal trace analysis, a... | 191 |
| [ts-debate-multimodal-collaborative-debate](skills/ts-debate-multimodal-collaborative-debate/SKILL.md) | Zero-shot time series reasoning via modality-specialized multi-agent debate. Assigns dedicated text, visual, and numeric... | 232 |
| [understanding-agent-scaling-llm-based](skills/understanding-agent-scaling-llm-based/SKILL.md) | Design diversity-aware multi-agent systems that maximize performance with fewer agents. Uses information-theoretic K* ef... | 204 |
| [valueflow-measuring-propagation-value](skills/valueflow-measuring-propagation-value/SKILL.md) | Measure and analyze how value perturbations propagate through multi-agent LLM systems using the ValueFlow framework. Dia... | 180 |
| [veri-sure-contract-aware-multi-agent-framework](skills/veri-sure-contract-aware-multi-agent-framework/SKILL.md) | Generate functionally correct RTL/Verilog code using a contract-aware multi-agent workflow with formal verification. Tri... | 186 |
| [villain-at-averimatec-verifying](skills/villain-at-averimatec-verifying/SKILL.md) | Build multi-agent fact-checking pipelines that verify image-text claims through modality-specific analysis, cross-modal ... | 248 |
| [visual-reasoning-over-time](skills/visual-reasoning-over-time/SKILL.md) | Analyze time series data using the MAS4TS Analyzer-Reasoner-Executor multi-agent paradigm: convert series to plots, extr... | 180 |
| [when-agents-misremember-collectively](skills/when-agents-misremember-collectively/SKILL.md) | Detect, measure, and defend against collective false-memory propagation (the Mandela Effect) in LLM multi-agent systems.... | 225 |
| [who-deserves-reward-sharp](skills/who-deserves-reward-sharp/SKILL.md) | Apply SHARP (Shapley-based credit attribution) to design and optimize multi-agent systems where each agent's individual ... | 220 |
| [yunque-deepresearch-technical-report](skills/yunque-deepresearch-technical-report/SKILL.md) | Hierarchical multi-agent deep research framework with dynamic context management and supervisor-based error recovery. Im... | 191 |

---

## Prompt Engineering

**123 skills**

| Skill | Description | Lines |
|-------|-------------|-------|
| [3-secbench-large-scale-evaluation-suite-security](skills/3-secbench-large-scale-evaluation-suite-security/SKILL.md) | Evaluate and harden LLM-based autonomous agents against adversarial attacks using the α³-SecBench layered security frame... | 182 |
| [addressing-explainability-generative-ai](skills/addressing-explainability-generative-ai/SKILL.md) | Explain generative AI outputs using the gSMILE perturbation-based attribution framework. Builds local surrogate models f... | 222 |
| [aegis-governance-integrity-security](skills/aegis-governance-integrity-security/SKILL.md) | Red-team and harden AI voice agents and LLM-powered service systems against adversarial misuse using the Aegis framework... | 237 |
| [agentark-distilling-multi-agent-intelligence](skills/agentark-distilling-multi-agent-intelligence/SKILL.md) | Distill multi-agent debate reasoning into a single LLM's behavior. Apply AgentArk's three-tier distillation strategy to ... | 199 |
| [agentdrive-open-benchmark-dataset](skills/agentdrive-open-benchmark-dataset/SKILL.md) | Generate structured autonomous driving scenarios and MCQ benchmarks using AgentDrive's factorized 7-axis prompt-to-JSON ... | 284 |
| [agentstepper-interactive-debugging-software](skills/agentstepper-interactive-debugging-software/SKILL.md) | Interactive debugging of LLM-powered software development agents using structured trajectory analysis, stepwise executio... | 197 |
| [alienlm-alienization-api-boundary-privacy](skills/alienlm-alienization-api-boundary-privacy/SKILL.md) | Implement AlienLM-style API-boundary privacy layers that protect sensitive text sent to black-box LLM APIs using vocabul... | 202 |
| [are-open-weight-ready-social](skills/are-open-weight-ready-social/SKILL.md) | Build LLM-based content moderation pipelines using zero-shot classification with open-weight models. Implements the stru... | 264 |
| [assessment-generative-named-entity](skills/assessment-generative-named-entity/SKILL.md) | Build generative NER systems using LLMs with optimal output formats and prompt engineering. Use when: 'extract entities ... | 245 |
| [attn-gs-attention-guided-context-compression](skills/attn-gs-attention-guided-context-compression/SKILL.md) | Compress long user contexts (profiles, histories, documents) into concise, high-quality summaries using attention-guided... | 158 |
| [automated-multiple-mini-interview](skills/automated-multiple-mini-interview/SKILL.md) | Multi-agent framework for scoring subjective, open-ended responses (interviews, essays, reflections) using transcript re... | 189 |
| [bayesflow-probability-inference-framework](skills/bayesflow-probability-inference-framework/SKILL.md) | Generate high-quality multi-step LLM workflows using Bayesian inference with parallel look-ahead rollouts and importance... | 204 |
| [benchmarking-zero-shot-few-shot-phishing](skills/benchmarking-zero-shot-few-shot-phishing/SKILL.md) | Detect phishing URLs using LLM zero-shot and few-shot prompting with structured classification prompts. Use when: 'class... | 215 |
| [beyond-confidence-rhythms-reasoning](skills/beyond-confidence-rhythms-reasoning/SKILL.md) | Analyze and improve LLM prompt robustness using the Token Constraint Bound (delta-TCB) metric from the paper 'Beyond Con... | 169 |
| [beyond-holistic-scores-automatic](skills/beyond-holistic-scores-automatic/SKILL.md) | Build trait-based essay scoring systems that evaluate argumentative writing across multiple rubric dimensions (Content, ... | 201 |
| [beyond-instrumental-substitutive-paradigms](skills/beyond-instrumental-substitutive-paradigms/SKILL.md) | Audit and diagnose cultural bias artifacts in LLM-powered applications using the Machine Culture framework. Detects Cult... | 189 |
| [beyond-prompting-robust-contextual](skills/beyond-prompting-robust-contextual/SKILL.md) | > | 220 |
| [bi-directional-bias-attribution-debiasing](skills/bi-directional-bias-attribution-debiasing/SKILL.md) | Detect and mitigate social biases in LLM outputs using neuron-level attribution and intervention, without modifying prom... | 184 |
| [blind-gods-broken-screens](skills/blind-gods-broken-screens/SKILL.md) | Architect secure, intent-centric agent systems using the Aura pattern: Hub-and-Spoke agent topology, cryptographic ident... | 219 |
| [breaking-protocol-security-analysis](skills/breaking-protocol-security-analysis/SKILL.md) | Audit and harden Model Context Protocol (MCP) server deployments against protocol-level vulnerabilities including capabi... | 264 |
| [bridging-modality-gap-roadside](skills/bridging-modality-gap-roadside/SKILL.md) | Build training-free pipelines that convert sparse 3D LiDAR point clouds into depth-encoded 2D images for classification ... | 211 |
| [c-mop-integrating-momentum-boundary-aware](skills/c-mop-integrating-momentum-boundary-aware/SKILL.md) | Optimize LLM system prompts iteratively using boundary-aware contrastive sampling and momentum-guided clustering from th... | 179 |
| [c3box-clip-based-class-incremental-learning](skills/c3box-clip-based-class-incremental-learning/SKILL.md) | Set up, configure, and run CLIP-based class-incremental learning experiments using the C3Box toolbox. Supports 17 CIL al... | 215 |
| [can-small-handle-context-summarized](skills/can-small-handle-context-summarized/SKILL.md) | Build context-summarized multi-turn QA systems that let small language models (SLMs) handle customer-service dialogues w... | 254 |
| [can-truly-embody-human](skills/can-truly-embody-human/SKILL.md) | Evaluate and improve personality-behavior alignment in LLM simulations of human social interactions. Uses the BFI-IRP ev... | 206 |
| [causal-perspective-enhancing-jailbreak-attack](skills/causal-perspective-enhancing-jailbreak-attack/SKILL.md) | Apply causal analysis to LLM safety: identify direct causal drivers of jailbreaks using prompt feature decomposition, bu... | 175 |
| [computational-approach-visual-metonymy](skills/computational-approach-visual-metonymy/SKILL.md) | Generate and evaluate visual metonymy -- indirect visual representations that evoke concepts through associated cues rat... | 179 |
| [creditaudit-2textnd-dimension-evaluation](skills/creditaudit-2textnd-dimension-evaluation/SKILL.md) | Evaluate and select LLMs using CreditAudit's 2D framework: mean ability plus stability risk (fluctuation) across system ... | 211 |
| [curate-train-refine-closed-loop-agentic-framework-](skills/curate-train-refine-closed-loop-agentic-framework/SKILL.md) | Build lightweight text classifiers from zero labeled data using an agentic Curate-Train-Refine loop. An LLM generates sy... | 148 |
| [darl-encouraging-diverse-answers](skills/darl-encouraging-diverse-answers/SKILL.md) | Generate diverse, high-quality answer variants for open-ended tasks using DARL's bounded-diversity framework. Use when: ... | 286 |
| [data-centric-interpretability-llm-based-multi-agen](skills/data-centric-interpretability-llm-based-multi-agen/SKILL.md) | Analyze LLM agent behavior across training runs using Sparse Autoencoder (SAE) features and LLM-summarizer pipelines. Gr... | 189 |
| [dcopilot-generative-ai-empowered-policy](skills/dcopilot-generative-ai-empowered-policy/SKILL.md) | Build hybrid LLM + hypernetwork systems that generate control policies for dynamic environments. Uses LLM-based reward s... | 297 |
| [deepasmr-llm-based-zero-shot-asmr](skills/deepasmr-llm-based-zero-shot-asmr/SKILL.md) | Build zero-shot ASMR speech generation systems using a two-stage LLM + flow-matching pipeline that separates speaking st... | 227 |
| [e2pl-prompt-learning-incomplete](skills/e2pl-prompt-learning-incomplete/SKILL.md) | Design prompt-learning systems for incremental multi-view multi-label classification with missing data. Use when: 'handl... | 188 |
| [effgen-enabling-small-language](skills/effgen-enabling-small-language/SKILL.md) | Deploy and optimize small language models (SLMs) as autonomous agents using the effGen framework. Implements prompt comp... | 193 |
| [error-taxonomy-guided-prompt-optimization](skills/error-taxonomy-guided-prompt-optimization/SKILL.md) | | | 162 |
| [fraudshield-knowledge-graph-empowered](skills/fraudshield-knowledge-graph-empowered/SKILL.md) | Detect and defend against fraudulent content in LLM inputs using knowledge-graph-augmented analysis. Builds a fraud tact... | 269 |
| [from-assistant-double-agent](skills/from-assistant-double-agent/SKILL.md) | Security audit and hardening for personalized LLM-based agents against prompt injection, tool poisoning, and memory atta... | 230 |
| [from-code-centric-concept-centric-teaching](skills/from-code-centric-concept-centric-teaching/SKILL.md) | Generate LLM-assisted coding labs that teach concepts through 'Vibe Coding' — producing working code paired with mandato... | 269 |
| [from-prompt-response-goal-directed-systems](skills/from-prompt-response-goal-directed-systems/SKILL.md) | Design production-grade agentic AI architectures with separated cognition/execution layers, typed tool interfaces, multi... | 177 |
| [generalizable-interpretable-rf-fingerprinting](skills/generalizable-interpretable-rf-fingerprinting/SKILL.md) | Build RF fingerprinting systems that combine learnable 2D shapelets with pre-trained LLMs for wireless device authentica... | 168 |
| [gflowpo-generative-flow-network](skills/gflowpo-generative-flow-network/SKILL.md) | Optimize LLM prompts using GFlowPO's iterative generate-evaluate-refine loop with diversity-preserving exploration and d... | 171 |
| [gradingattack-attacking-short-answer](skills/gradingattack-attacking-short-answer/SKILL.md) | Audit LLM-based automatic short answer grading (ASAG) systems for adversarial vulnerabilities using token-level and prom... | 243 |
| [group-distributionally-robust-optimization-driven](skills/group-distributionally-robust-optimization-driven/SKILL.md) | Apply Group Distributionally Robust Optimization (GDRO) to RL-based LLM training. Dynamically classify prompts by diffic... | 205 |
| [gutenocr-grounded-vision-language-front-end](skills/gutenocr-grounded-vision-language-front-end/SKILL.md) | Build grounded OCR pipelines using GutenOCR's prompt-based interface for reading, detection, and spatial grounding on do... | 210 |
| [helios-hierarchical-graph-abstraction](skills/helios-hierarchical-graph-abstraction/SKILL.md) | Structure-aware binary decompilation using hierarchical control-flow graph abstraction for LLMs. Converts binary program... | 204 |
| [how-few-shot-demonstrations-affect](skills/how-few-shot-demonstrations-affect/SKILL.md) | Design prompt-based LLM safety defenses using optimal few-shot strategies. Applies the finding that few-shot demonstrati... | 199 |
| [icl-evader-zero-query-black-box-evasion](skills/icl-evader-zero-query-black-box-evasion/SKILL.md) | Harden ICL classification prompts against zero-query black-box evasion attacks. Audit in-context learning pipelines for ... | 251 |
| [icon-intent-context-coupling-multi-turn](skills/icon-intent-context-coupling-multi-turn/SKILL.md) | Build multi-turn LLM safety evaluation harnesses using the Intent-Context Coupling framework from ICON. Generates struct... | 243 |
| [instructtime-time-series-classification-multimodal](skills/instructtime-time-series-classification-multimodal/SKILL.md) | Reformulate time series classification as a multimodal generative task using LLMs. Discretizes time series into tokens, ... | 165 |
| [interpreting-controlling-behavior-constitutions](skills/interpreting-controlling-behavior-constitutions/SKILL.md) | Learn and apply natural-language constitutions that map prompt edits to predictable model behavior changes. Use atomic c... | 182 |
| [iterative-refinement-improves-compositional](skills/iterative-refinement-improves-compositional/SKILL.md) | Implement iterative critic-guided refinement loops for compositional image generation. Uses a VLM critic to progressivel... | 199 |
| [knowledge-restoration-driven-prompt-optimization](skills/knowledge-restoration-driven-prompt-optimization/SKILL.md) | | | 211 |
| [large-geolocation-extraction-humanitarian](skills/large-geolocation-extraction-humanitarian/SKILL.md) | Extract and geocode location mentions from humanitarian and crisis texts using a two-step LLM pipeline: few-shot NER for... | 213 |
| [large-reasoning-failures](skills/large-reasoning-failures/SKILL.md) | Detect and mitigate known LLM reasoning failures during code generation, review, and problem-solving. Applies the taxono... | 218 |
| [less-noise-more-voice](skills/less-noise-more-voice/SKILL.md) | Identify and remove interference tokens from prompts to improve LLM reasoning accuracy. Based on the LENS framework (Les... | 236 |
| [leveraging-turkish-skill-extraction](skills/leveraging-turkish-skill-extraction/SKILL.md) | Extract and normalize skills from job postings using a two-stage LLM pipeline: dynamic few-shot skill identification fol... | 198 |
| [lhaw-controllable-underspecification-long-horizon](skills/lhaw-controllable-underspecification-long-horizon/SKILL.md) | Detect and handle ambiguity in long-horizon agent tasks using the LHAW framework. Systematically identify underspecified... | 170 |
| [lingua-safetybench-benchmark-safety-evaluation-mul](skills/lingua-safetybench-benchmark-safety-evaluation-mul/SKILL.md) | Evaluate and stress-test multilingual vision-language model safety using the Lingua-SafetyBench methodology. Constructs ... | 160 |
| [llm-based-sql-generation-prompting](skills/llm-based-sql-generation-prompting/SKILL.md) | Generate accurate SQL from natural language using the SSEV pipeline: schema-linked prompting, execution-guided self-refi... | 174 |
| [llm-not-all-you](skills/llm-not-all-you/SKILL.md) | Systematic model selection advisor for classification tasks — chooses between classical ML, zero-shot LLMs/VLMs, and fin... | 187 |
| [llm-prompt-evaluation-educational](skills/llm-prompt-evaluation-educational/SKILL.md) | Systematically design, evaluate, and rank LLM prompts for educational applications using tournament-style Glicko-2 compa... | 223 |
| [lmmrec-llm-driven-motivation-aware-multimodal](skills/lmmrec-llm-driven-motivation-aware-multimodal/SKILL.md) | Build motivation-aware recommendation systems that use LLM chain-of-thought prompting to extract user and item motivatio... | 217 |
| [mitigating-conversational-inertia-multi-turn](skills/mitigating-conversational-inertia-multi-turn/SKILL.md) | Detect and break conversational inertia in multi-turn agent interactions — where an LLM repeats its own prior actions as... | 236 |
| [mpib-benchmark-medical-prompt](skills/mpib-benchmark-medical-prompt/SKILL.md) | Evaluate and defend clinical LLM systems against prompt injection attacks using the MPIB benchmark methodology. Implemen... | 177 |
| [mrag-benchmarking-retrieval-augmented-generation](skills/mrag-benchmarking-retrieval-augmented-generation/SKILL.md) | Build and evaluate biomedical RAG pipelines using the MRAG benchmark methodology. Configures retrieval, prompting, and g... | 183 |
| [multi-agentic-ai-fairness-aware-accelerated](skills/multi-agentic-ai-fairness-aware-accelerated/SKILL.md) | Design and implement multi-agent systems for fairness-aware, low-latency inference orchestration across distributed edge... | 200 |
| [multimodal-fine-tuning-synthetic-captions](skills/multimodal-fine-tuning-synthetic-captions/SKILL.md) | Generate synthetic image captions with MLLMs and fine-tune CLIP models using multimodal objectives with supervised contr... | 201 |
| [mulvul-retrieval-augmented-multi-agent-code](skills/mulvul-retrieval-augmented-multi-agent-code/SKILL.md) | Multi-agent vulnerability detection using coarse-to-fine routing, contrastive retrieval, and cross-model prompt evolutio... | 204 |
| [naamse-framework-evolutionary-security](skills/naamse-framework-evolutionary-security/SKILL.md) | Implement evolutionary security evaluation for AI agents using the NAAMSE framework — genetic prompt mutation, hierarchi... | 201 |
| [noir-privacy-preserving-generation-code](skills/noir-privacy-preserving-generation-code/SKILL.md) | Design and implement privacy-preserving code generation systems using the NOIR split-architecture pattern: client-side e... | 223 |
| [open-tutorai-open-source-platform](skills/open-tutorai-open-source-platform/SKILL.md) | Build personalized AI tutoring systems with structured onboarding, four-layer prompt architecture, adaptive lesson gener... | 266 |
| [optimizing-prompts-causal-approach](skills/optimizing-prompts-causal-approach/SKILL.md) | Optimize LLM prompts using causal inference (CPO). Isolates true prompt effectiveness from query difficulty via Double M... | 156 |
| [pand-prompt-aware-neighborhood-distillation](skills/pand-prompt-aware-neighborhood-distillation/SKILL.md) | Implement PAND (Prompt-Aware Neighborhood Distillation) for distilling Vision-Language Models into lightweight networks ... | 189 |
| [parse-open-domain-reasoning-question](skills/parse-open-domain-reasoning-question/SKILL.md) | Build and evaluate reasoning-focused QA systems for low-resource languages using the PARSE methodology: structured promp... | 220 |
| [personality-as-relational-infrastructure](skills/personality-as-relational-infrastructure/SKILL.md) | Design LLM messaging systems that infuse Big Five personality traits for sustained user engagement. Uses aggregate-expos... | 252 |
| [physical-prompt-injection-attacks](skills/physical-prompt-injection-attacks/SKILL.md) | > | 291 |
| [predicting-intermittent-job-failure](skills/predicting-intermittent-job-failure/SKILL.md) | Classify and diagnose intermittent CI/CD job failures from execution logs using the FlaXifyer few-shot approach and LogS... | 280 |
| [prompt-augmentation-scales-up](skills/prompt-augmentation-scales-up/SKILL.md) | | | 274 |
| [prompt-driven-development-claude](skills/prompt-driven-development-claude/SKILL.md) | | | 188 |
| [prompt-injection-attacks-agentic](skills/prompt-injection-attacks-agentic/SKILL.md) | > | 207 |
| [promptrl-prompt-matters-rl](skills/promptrl-prompt-matters-rl/SKILL.md) | Implement PromptRL-style joint prompt-refinement + RL training loops for flow-based image generation. Use when the user ... | 196 |
| [raca-representation-aware-coverage-criteria](skills/raca-representation-aware-coverage-criteria/SKILL.md) | Evaluate and improve LLM safety test suites using representation-aware coverage criteria. Implements the RACA framework ... | 242 |
| [raicl-retrieval-augmented-in-context-learning](skills/raicl-retrieval-augmented-in-context-learning/SKILL.md) | Build retrieval-augmented in-context learning (RAICL) pipelines that convert time-series or signal data into images and ... | 228 |
| [rapo-risk-aware-preference-optimization](skills/rapo-risk-aware-preference-optimization/SKILL.md) | Apply risk-aware preference optimization to make LLM reasoning chains safer against jailbreak attacks. Implements adapti... | 203 |
| [rc-grpo-reward-conditioned-group-relative](skills/rc-grpo-reward-conditioned-group-relative/SKILL.md) | Implement reward-conditioned training pipelines for multi-turn tool-calling agents using RC-GRPO. Injects discrete rewar... | 228 |
| [rebel-hidden-knowledge-recovery](skills/rebel-hidden-knowledge-recovery/SKILL.md) | Machine unlearning for LLMs aims to remove sensitive or copyrighted data from trained models. Implements techniques from... | 28 |
| [redvisor-reasoning-aware-prompt-injection](skills/redvisor-reasoning-aware-prompt-injection/SKILL.md) | Defend LLM applications against prompt injection using RedVisor's two-phase reasoning-then-responding architecture. Impl... | 223 |
| [remedit-diffusion-editing-riemannian](skills/remedit-diffusion-editing-riemannian/SKILL.md) | Implement Riemannian-geometry-based diffusion image editing pipelines using geodesic latent navigation, dual-SLERP blend... | 244 |
| [reprompt-prompt-generation-intelligent](skills/reprompt-prompt-generation-intelligent/SKILL.md) | Generate optimized system and user prompts for coding agents using requirements engineering principles from the REprompt... | 191 |
| [resagent-entropy-based-prior-point](skills/resagent-entropy-based-prior-point/SKILL.md) | Implement entropy-guided coarse-to-fine visual grounding pipelines for referring expression segmentation and point-promp... | 266 |
| [rethinking-llm-as-a-judge-representation-as-a-judg](skills/rethinking-llm-as-a-judge-representation-as-a-judg/SKILL.md) | Build probing-based evaluation pipelines that judge LLM output quality using hidden-state representations from small lan... | 160 |
| [sdr-cir-semantic-debias-retrieval](skills/sdr-cir-semantic-debias-retrieval/SKILL.md) | Build training-free composed image retrieval systems that combine a reference image with modification text to find targe... | 171 |
| [self-hinting-enhance-reinforcement-learning](skills/self-hinting-enhance-reinforcement-learning/SKILL.md) | Apply the SAGE self-hinting technique to improve LLM problem-solving by generating graduated hints that boost solution d... | 174 |
| [sicl-at-another-way-adapt](skills/sicl-at-another-way-adapt/SKILL.md) | Adapt auditory LLMs to low-resource speech/audio tasks using Speech In-Context Learning Adaptation Training (SICL-AT). S... | 168 |
| [skillrl-evolving-agents-recursive](skills/skillrl-evolving-agents-recursive/SKILL.md) | Build self-improving agent systems that distill raw execution traces into a hierarchical skill library (SkillBank) and r... | 184 |
| [sogk-one-token-explicit-graph](skills/sogk-one-token-explicit-graph/SKILL.md) | Represent graph topology as a single discrete token (<SOG_k>) for LLM reasoning, replacing verbose graph verbalization. ... | 128 |
| [spider-sense-intrinsic-risk-sensing](skills/spider-sense-intrinsic-risk-sensing/SKILL.md) | Implement event-driven, hierarchical security screening for LLM agent systems using Intrinsic Risk Sensing. Adds latent ... | 212 |
| [steer2adapt-dynamically-composing-steering](skills/steer2adapt-dynamically-composing-steering/SKILL.md) | Implement the Steer2Adapt framework for adapting LLMs at inference time by dynamically composing steering vectors from a... | 209 |
| [system-name-address-parsing](skills/system-name-address-parsing/SKILL.md) | Parse unstructured person names and addresses into a structured 17-field schema using prompt-driven extraction with laye... | 207 |
| [task-oriented-robot-human-handovers-legged](skills/task-oriented-robot-human-handovers-legged/SKILL.md) | Implement task-oriented robot-to-human object handover systems using LLM-driven affordance reasoning and exemplar-based ... | 261 |
| [the-compliance-paradox-semantic-instruction](skills/the-compliance-paradox-semantic-instruction/SKILL.md) | Detect and defend against adversarial prompt injections hidden in code submissions that exploit LLM instruction-followin... | 229 |
| [the-landscape-prompt-injection](skills/the-landscape-prompt-injection/SKILL.md) | Harden LLM agent systems against prompt injection using layered text/model/execution defenses and the AgentPI evaluation... | 244 |
| [the-necessity-unified-framework](skills/the-necessity-unified-framework/SKILL.md) | Design and implement standardized, reproducible evaluation harnesses for LLM-based agents. Eliminates confounding factor... | 185 |
| [topt-task-oriented-prompt-tuning](skills/topt-task-oriented-prompt-tuning/SKILL.md) | Apply Task-Oriented Prompt Tuning (ToPT) to build urban region representation learning pipelines that combine spatial-aw... | 184 |
| [tracellm-leveraging-prompt-engineering](skills/tracellm-leveraging-prompt-engineering/SKILL.md) | Establish and verify traceability links between software artifacts (requirements, design docs, test cases, regulations) ... | 179 |
| [tracenas-zero-shot-pruning-gradient](skills/tracenas-zero-shot-pruning-gradient/SKILL.md) | Implement TraceNAS-style zero-shot LLM structured pruning using gradient trace correlation as a scale-invariant proxy. J... | 226 |
| [trailblazer-history-guided-reinforcement-learning](skills/trailblazer-history-guided-reinforcement-learning/SKILL.md) | Build history-aware RL pipelines for multi-turn LLM red-teaming and safety evaluation. Implements attention-weighted int... | 244 |
| [ts-debate-multimodal-collaborative-debate](skills/ts-debate-multimodal-collaborative-debate/SKILL.md) | Zero-shot time series reasoning via modality-specialized multi-agent debate. Assigns dedicated text, visual, and numeric... | 232 |
| [uncertainty-and-fairness-awareness](skills/uncertainty-and-fairness-awareness/SKILL.md) | Audit LLM-based recommendation systems for predictive uncertainty and demographic fairness bias. Implements the SNSR/SNS... | 230 |
| [unikie-bench-benchmarking-multimodal-key](skills/unikie-bench-benchmarking-multimodal-key/SKILL.md) | Extract structured key information from document images using schema-guided prompting for LMMs. Builds KIE pipelines tha... | 292 |
| [usage-effects-requirements-ai-coding](skills/usage-effects-requirements-ai-coding/SKILL.md) | Optimize AI coding assistant interactions using empirical enterprise findings on usage patterns, productivity factors, a... | 228 |
| [v0-generalist-value-any-policy](skills/v0-generalist-value-any-policy/SKILL.md) | Implement V0-style generalist value estimation that profiles any LLM policy from behavioral history rather than paramete... | 169 |
| [vidvec-unlocking-video-mllm](skills/vidvec-unlocking-video-mllm/SKILL.md) | Extract high-quality video-text embeddings from generative MLLMs using intermediate-layer representations and text-only ... | 197 |
| [viola-video-in-context-learning](skills/viola-video-in-context-learning/SKILL.md) | Apply the VIOLA framework for label-efficient in-context learning on video or multimodal data. Uses density-uncertainty-... | 221 |
| [vowelprompt-hearing-speech-emotions](skills/vowelprompt-hearing-speech-emotions/SKILL.md) | Build speech emotion recognition pipelines that augment LLMs with vowel-level prosodic features converted to natural lan... | 287 |
| [voxmorph-scalable-zero-shot-voice](skills/voxmorph-scalable-zero-shot-voice/SKILL.md) | Build and deploy zero-shot voice identity morphing pipelines using disentangled prosody/timbre embeddings and Spherical ... | 203 |
| [wavlink-compact-audio-text-embeddings](skills/wavlink-compact-audio-text-embeddings/SKILL.md) | Build compact audio-text embedding systems using WavLink's global Whisper token architecture with Matryoshka dimensional... | 197 |
| [when-agents-misremember-collectively](skills/when-agents-misremember-collectively/SKILL.md) | Detect, measure, and defend against collective false-memory propagation (the Mandela Effect) in LLM multi-agent systems.... | 225 |
| [when-better-prompts-hurt](skills/when-better-prompts-hurt/SKILL.md) | Evaluation-driven prompt iteration using the Define-Test-Diagnose-Fix loop and Minimum Viable Evaluation Suite (MVES). P... | 196 |
| [whispers-wealth-red-teaming-googles](skills/whispers-wealth-red-teaming-googles/SKILL.md) | Red-team LLM-based agentic payment systems against prompt injection attacks targeting transaction integrity and credenti... | 218 |
| [zero-shot-product-attribute-labeling](skills/zero-shot-product-attribute-labeling/SKILL.md) | Extract and classify product attributes from images using Vision-Language Models with structured prompts and a three-tie... | 268 |
| [zero2text-zero-training-cross-domain-inversion](skills/zero2text-zero-training-cross-domain-inversion/SKILL.md) | Implement embedding inversion attacks that reconstruct original text from vector embeddings without training data, using... | 188 |

---

## Security & Safety

**116 skills**

| Skill | Description | Lines |
|-------|-------------|-------|
| [3-secbench-large-scale-evaluation-suite-security](skills/3-secbench-large-scale-evaluation-suite-security/SKILL.md) | Evaluate and harden LLM-based autonomous agents against adversarial attacks using the α³-SecBench layered security frame... | 182 |
| [aegis-governance-integrity-security](skills/aegis-governance-integrity-security/SKILL.md) | Red-team and harden AI voice agents and LLM-powered service systems against adversarial misuse using the Aegis framework... | 237 |
| [agent-fence-mapping-security-vulnerabilities](skills/agent-fence-mapping-security-vulnerabilities/SKILL.md) | > | 211 |
| [agent2agent-threats-safety-critical-assistants](skills/agent2agent-threats-safety-critical-assistants/SKILL.md) | Threat model multi-agent LLM systems using the AgentHeLLM framework -- formally separating asset identification from att... | 207 |
| [agentdog-diagnostic-guardrail-framework](skills/agentdog-diagnostic-guardrail-framework/SKILL.md) | > | 340 |
| [agentdrive-open-benchmark-dataset](skills/agentdrive-open-benchmark-dataset/SKILL.md) | Generate structured autonomous driving scenarios and MCQ benchmarks using AgentDrive's factorized 7-axis prompt-to-JSON ... | 284 |
| [agenticsimlaw-juvenile-courtroom-multi-agent](skills/agenticsimlaw-juvenile-courtroom-multi-agent/SKILL.md) | Structured multi-agent courtroom debate for explainable high-stakes tabular decisions. Use when: 'set up a multi-agent d... | 181 |
| [agenttrace-structured-logging-framework](skills/agenttrace-structured-logging-framework/SKILL.md) | Implement structured, multi-surface observability logging for LLM agent systems using the AgentTrace pattern: operationa... | 152 |
| [aicd-bench-challenging-benchmark](skills/aicd-bench-challenging-benchmark/SKILL.md) | Detect whether source code was written by a human or generated by an AI model, attribute code to specific model families... | 211 |
| [an-cost-efficient-agentic-framework](skills/an-cost-efficient-agentic-framework/SKILL.md) | Audit Ethereum smart contracts for business logic vulnerabilities using Heimdallr's four-phase agentic pipeline: functio... | 202 |
| [benchmarking-zero-shot-few-shot-phishing](skills/benchmarking-zero-shot-few-shot-phishing/SKILL.md) | Detect phishing URLs using LLM zero-shot and few-shot prompting with structured classification prompts. Use when: 'class... | 215 |
| [biasscope-automated-detection-bias](skills/biasscope-automated-detection-bias/SKILL.md) | Automatically discover and test for hidden biases in LLM-as-a-Judge evaluation pipelines using the BiasScope framework. ... | 206 |
| [binaryppo-policy-optimization-binary](skills/binaryppo-policy-optimization-binary/SKILL.md) | Implement BinaryPPO, an offline RL framework that reformulates binary classification as reward maximization with confide... | 173 |
| [blind-gods-broken-screens](skills/blind-gods-broken-screens/SKILL.md) | Architect secure, intent-centric agent systems using the Aura pattern: Hub-and-Spoke agent topology, cryptographic ident... | 219 |
| [breaking-protocol-security-analysis](skills/breaking-protocol-security-analysis/SKILL.md) | Audit and harden Model Context Protocol (MCP) server deployments against protocol-level vulnerabilities including capabi... | 264 |
| [causal-perspective-enhancing-jailbreak-attack](skills/causal-perspective-enhancing-jailbreak-attack/SKILL.md) | Apply causal analysis to LLM safety: identify direct causal drivers of jailbreaks using prompt feature decomposition, bu... | 175 |
| [co-redteam-orchestrated-security-discovery](skills/co-redteam-orchestrated-security-discovery/SKILL.md) | Multi-agent security vulnerability discovery and exploitation using Co-RedTeam's orchestrated workflow. Decomposes secur... | 197 |
| [constitutional-spec-driven-development-enforcing](skills/constitutional-spec-driven-development-enforcing/SKILL.md) | Enforce security by construction in AI-generated code using Constitutional Spec-Driven Development (CSDD). Creates a ver... | 258 |
| [constructing-multi-label-hierarchical-classificati](skills/constructing-multi-label-hierarchical-classificati/SKILL.md) | Build multi-label hierarchical classifiers for MITRE ATT&CK text tagging using stage-wise classical ML (SGD-SVM + TF-IDF... | 226 |
| [context-sensitive-pointer-analysis-arkts](skills/context-sensitive-pointer-analysis-arkts/SKILL.md) | Perform context-sensitive pointer analysis for ArkTS/TypeScript code targeting OpenHarmony. Build precise call graphs, r... | 223 |
| [cutting-gordian-knot-detecting](skills/cutting-gordian-knot-detecting/SKILL.md) | Detect malicious PyPI/NPM packages using behavioral pattern mining and semantic reasoning (PyGuard). Use when: 'scan thi... | 212 |
| [david-vs-goliath-verifiable](skills/david-vs-goliath-verifiable/SKILL.md) | Audit and harden tool-augmented AI agent systems against Tag-Along Attacks -- adversarial agent-to-agent jailbreaks that... | 165 |
| [do-vlms-have-moral](skills/do-vlms-have-moral/SKILL.md) | Audit and harden the moral robustness of Vision-Language Model (VLM) pipelines against adversarial perturbations that fl... | 211 |
| [draincode-stealthy-energy-consumption](skills/draincode-stealthy-energy-consumption/SKILL.md) | Evaluate and defend RAG-based code generation systems against energy-drain attacks that poison retrieval contexts to inf... | 221 |
| [drugr-optimizing-molecular-drugs](skills/drugr-optimizing-molecular-drugs/SKILL.md) | Optimize molecular drug candidates using LLM-based explicit pharmacological reasoning over SMILES structures. Applies th... | 187 |
| [efficient-adaptable-detection-malicious](skills/efficient-adaptable-detection-malicious/SKILL.md) | > | 298 |
| [evaluating-enhancing-vulnerability-reasoning](skills/evaluating-enhancing-vulnerability-reasoning/SKILL.md) | Perform DAG-structured vulnerability reasoning on code, modeling causal dependencies between code facts instead of linea... | 211 |
| [extracting-recurring-vulnerabilities-black-box](skills/extracting-recurring-vulnerabilities-black-box/SKILL.md) | > | 188 |
| [following-dragons-code-review-guided](skills/following-dragons-code-review-guided/SKILL.md) | Extract security-relevant signals from code review comments and translate them into fuzzer-guiding annotations using the... | 158 |
| [from-assistant-double-agent](skills/from-assistant-double-agent/SKILL.md) | Security audit and hardening for personalized LLM-based agents against prompt injection, tool poisoning, and memory atta... | 230 |
| [from-data-behavior-predicting](skills/from-data-behavior-predicting/SKILL.md) | Predict unintended LLM behaviors (bias, safety regressions) from training data BEFORE fine-tuning, using the MDF (Manipu... | 209 |
| [from-detection-prevention-explaining](skills/from-detection-prevention-explaining/SKILL.md) | Proactively identify security-critical code regions and generate prevention-oriented explanations before vulnerabilities... | 265 |
| [from-helpfulness-toxic-proactivity](skills/from-helpfulness-toxic-proactivity/SKILL.md) | Diagnose and mitigate Toxic Proactivity in LLM agent systems -- the failure mode where agents override ethical constrain... | 194 |
| [from-sparse-decisions-dense](skills/from-sparse-decisions-dense/SKILL.md) | Build content moderation and safety classification systems using multi-attribute trajectory reasoning instead of binary ... | 261 |
| [gamms-graph-based-adversarial](skills/gamms-graph-based-adversarial/SKILL.md) | > | 305 |
| [gavel-rule-based-safety-activation](skills/gavel-rule-based-safety-activation/SKILL.md) | Implement GAVEL-style rule-based activation safety monitoring for LLMs using Cognitive Elements (CEs) and Boolean predic... | 194 |
| [gradingattack-attacking-short-answer](skills/gradingattack-attacking-short-answer/SKILL.md) | Audit LLM-based automatic short answer grading (ASAG) systems for adversarial vulnerabilities using token-level and prom... | 243 |
| [hallucination-resistant-security-planning](skills/hallucination-resistant-security-planning/SKILL.md) | > | 259 |
| [how-few-shot-demonstrations-affect](skills/how-few-shot-demonstrations-affect/SKILL.md) | Design prompt-based LLM safety defenses using optimal few-shot strategies. Applies the finding that few-shot demonstrati... | 199 |
| [humans-welcome-observe-first-look](skills/humans-welcome-observe-first-look/SKILL.md) | Analyze AI agent social network activity using topic taxonomy classification and multi-level toxicity scoring. Detects c... | 191 |
| [icl-evader-zero-query-black-box-evasion](skills/icl-evader-zero-query-black-box-evasion/SKILL.md) | Harden ICL classification prompts against zero-query black-box evasion attacks. Audit in-context learning pipelines for ... | 251 |
| [icon-intent-context-coupling-multi-turn](skills/icon-intent-context-coupling-multi-turn/SKILL.md) | Build multi-turn LLM safety evaluation harnesses using the Intent-Context Coupling framework from ICON. Generates struct... | 243 |
| [infa-guard-mitigating-malicious-propagation](skills/infa-guard-mitigating-malicious-propagation/SKILL.md) | > | 262 |
| [jailbreaking-calibration](skills/jailbreaking-calibration/SKILL.md) | > | 211 |
| [jailbreaks-vision-multimodal-reasoning](skills/jailbreaks-vision-multimodal-reasoning/SKILL.md) | > | 250 |
| [kid-knowledge-injected-dual-head-learning](skills/kid-knowledge-injected-dual-head-learning/SKILL.md) | Build knowledge-grounded multimodal content classifiers using the KID dual-head architecture: entity-anchored knowledge ... | 185 |
| [lingua-safetybench-benchmark-safety-evaluation-mul](skills/lingua-safetybench-benchmark-safety-evaluation-mul/SKILL.md) | Evaluate and stress-test multilingual vision-language model safety using the Lingua-SafetyBench methodology. Constructs ... | 160 |
| [llama-31-foundationai-securityllm-reasoning-8b-tec](skills/llama-31-foundationai-securityllm-reasoning-8b-tec/SKILL.md) | > | 251 |
| [lost-translation-comparative-study](skills/lost-translation-comparative-study/SKILL.md) | Cross-lingual safety evaluation for LLMs using the CompositeHarm methodology. Builds multilingual safety benchmarks that... | 176 |
| [lps-bench-benchmarking-safety-awareness](skills/lps-bench-benchmarking-safety-awareness/SKILL.md) | > | 191 |
| [malicious-agent-skills-wild](skills/malicious-agent-skills-wild/SKILL.md) | > | 264 |
| [malicious-repurposing-open-science](skills/malicious-repurposing-open-science/SKILL.md) | | | 212 |
| [mhdash-online-platform-benchmarking](skills/mhdash-online-platform-benchmarking/SKILL.md) | Build risk-aware evaluation pipelines for mental health AI assistants using the MHDash framework. Implements multi-dimen... | 285 |
| [mind-ambiguity-aleatoric-uncertainty](skills/mind-ambiguity-aleatoric-uncertainty/SKILL.md) | Detect ambiguous user queries in safety-critical QA systems using aleatoric uncertainty probes on LLM hidden states, the... | 225 |
| [mpib-benchmark-medical-prompt](skills/mpib-benchmark-medical-prompt/SKILL.md) | Evaluate and defend clinical LLM systems against prompt injection attacks using the MPIB benchmark methodology. Implemen... | 177 |
| [multi-agent-end-to-end-vulnerability-management](skills/multi-agent-end-to-end-vulnerability-management/SKILL.md) | Detect, confirm, repair, and validate recurring software vulnerabilities using a multi-agent pipeline modeled on MAVM. B... | 196 |
| [multi-targeted-graph-backdoor-attack](skills/multi-targeted-graph-backdoor-attack/SKILL.md) | Implement and analyze multi-targeted backdoor attacks on Graph Neural Networks (GNNs) using subgraph injection. Use when... | 193 |
| [multimodal-multi-agent-ransomware-analysis](skills/multimodal-multi-agent-ransomware-analysis/SKILL.md) | Build multimodal multi-agent pipelines for ransomware classification using specialized per-modality agents, autoencoder ... | 261 |
| [mulvul-retrieval-augmented-multi-agent-code](skills/mulvul-retrieval-augmented-multi-agent-code/SKILL.md) | Multi-agent vulnerability detection using coarse-to-fine routing, contrastive retrieval, and cross-model prompt evolutio... | 204 |
| [naamse-framework-evolutionary-security](skills/naamse-framework-evolutionary-security/SKILL.md) | Implement evolutionary security evaluation for AI agents using the NAAMSE framework — genetic prompt mutation, hierarchi... | 201 |
| [neurofilter-privacy-guardrails-conversational](skills/neurofilter-privacy-guardrails-conversational/SKILL.md) | > | 288 |
| [noir-privacy-preserving-generation-code](skills/noir-privacy-preserving-generation-code/SKILL.md) | Design and implement privacy-preserving code generation systems using the NOIR split-architecture pattern: client-side e... | 223 |
| [now-you-hear-me](skills/now-you-hear-me/SKILL.md) | Audit and defend large audio-language models (LALMs) against narrative-style audio jailbreaks. Based on the 'Now You Hea... | 311 |
| [omni-safety-under-cross-modality-conflict](skills/omni-safety-under-cross-modality-conflict/SKILL.md) | Audit and harden omni-modal LLM safety against cross-modality attacks using refusal-vector analysis and OmniSteer alignm... | 217 |
| [persona-jailbreaking](skills/persona-jailbreaking/SKILL.md) | Audit and defend LLM-powered applications against persona manipulation attacks using the PHISH framework (Persona Hijack... | 301 |
| [physical-prompt-injection-attacks](skills/physical-prompt-injection-attacks/SKILL.md) | > | 291 |
| [privacy-collapse-benign-fine-tuning](skills/privacy-collapse-benign-fine-tuning/SKILL.md) | Audit fine-tuning datasets and pipelines for privacy collapse — the silent failure where benign training data degrades a... | 195 |
| [prompt-injection-attacks-agentic](skills/prompt-injection-attacks-agentic/SKILL.md) | > | 207 |
| [proopf-benchmarking-improving-professional-grade](skills/proopf-benchmarking-improving-professional-grade/SKILL.md) | Translate natural-language power system operational requirements into executable Optimal Power Flow (OPF) optimization c... | 218 |
| [protoken-token-level-attribution-federated](skills/protoken-token-level-attribution-federated/SKILL.md) | Implement ProToken-style token-level attribution to trace which federated learning client(s) contributed to each generat... | 164 |
| [qrs-rule-synthesizing-neuro-symbolic-triad](skills/qrs-rule-synthesizing-neuro-symbolic-triad/SKILL.md) | Autonomous vulnerability discovery using the QRS (Query, Review, Sanitize) neuro-symbolic triad. Generates CodeQL querie... | 229 |
| [r1-syntheticvl-synthetic-data-generative](skills/r1-syntheticvl-synthetic-data-generative/SKILL.md) | Synthesize high-quality multimodal training data using Collective Adversarial Data Synthesis (CADS). Implements a cyclic... | 226 |
| [raca-representation-aware-coverage-criteria](skills/raca-representation-aware-coverage-criteria/SKILL.md) | Evaluate and improve LLM safety test suites using representation-aware coverage criteria. Implements the RACA framework ... | 242 |
| [ral-bench-benchmarking-application-level-functiona](skills/ral-bench-benchmarking-application-level-functiona/SKILL.md) | Generate and evaluate complete multi-file application repositories with both functional correctness and non-functional q... | 179 |
| [rapid-real-time-deterministic-trajectory](skills/rapid-real-time-deterministic-trajectory/SKILL.md) | Distill diffusion-based trajectory planners into fast deterministic policies using score-regularized optimization and sa... | 185 |
| [rapo-risk-aware-preference-optimization](skills/rapo-risk-aware-preference-optimization/SKILL.md) | Apply risk-aware preference optimization to make LLM reasoning chains safer against jailbreak attacks. Implements adapti... | 203 |
| [redsage-cybersecurity-generalist](skills/redsage-cybersecurity-generalist/SKILL.md) | | | 234 |
| [redvisor-reasoning-aware-prompt-injection](skills/redvisor-reasoning-aware-prompt-injection/SKILL.md) | Defend LLM applications against prompt injection using RedVisor's two-phase reasoning-then-responding architecture. Impl... | 223 |
| [reinforcement-learning-backtracking-feedback](skills/reinforcement-learning-backtracking-feedback/SKILL.md) | Implement RLBF (Reinforcement Learning with Backtracking Feedback) for LLM safety — a framework where models learn to de... | 289 |
| [reverse-engineering-editing](skills/reverse-engineering-editing/SKILL.md) | Audit and defend locate-then-edit model editing methods (ROME, MEMIT, AlphaEdit) against data leakage via spectral analy... | 182 |
| [reward-free-alignment-conflicting-objectives](skills/reward-free-alignment-conflicting-objectives/SKILL.md) | Implement multi-objective LLM alignment using RACO (Reward-free Alignment for Conflicting Objectives) — a method that re... | 239 |
| [risk-awareness-injection-calibrating](skills/risk-awareness-injection-calibrating/SKILL.md) | Implement Risk Awareness Injection (RAI) to defend vision-language models against multimodal jailbreak attacks without r... | 244 |
| [rvb-automating-ai-system](skills/rvb-automating-ai-system/SKILL.md) | Harden code and AI guardrails through iterative Red Team vs Blue Team adversarial games. Use when the user says 'harden ... | 222 |
| [rvcbench-benchmarking-robustness-voice](skills/rvcbench-benchmarking-robustness-voice/SKILL.md) | Benchmark and harden voice cloning systems against real-world degradation using the RVCBench framework. Evaluates VC mod... | 164 |
| [safepred-predictive-guardrail-computer-using](skills/safepred-predictive-guardrail-computer-using/SKILL.md) | Implement predictive safety guardrails for computer-using agents and automated pipelines using world-model-based risk pr... | 179 |
| [seccodeprm-process-reward-code](skills/seccodeprm-process-reward-code/SKILL.md) | Step-level security scoring for code generation and vulnerability detection using process reward model techniques. Use w... | 201 |
| [selective-steering-norm-preserving-control](skills/selective-steering-norm-preserving-control/SKILL.md) | Implement norm-preserving activation steering for LLMs using discriminative layer selection and Givens rotation. Use whe... | 174 |
| [self-improving-pretraining-post-trained-pretrain](skills/self-improving-pretraining-post-trained-pretrain/SKILL.md) | Build data curation pipelines that use a strong post-trained model to score, filter, and rewrite pretraining corpora for... | 256 |
| [sifting-noise-comparative-study](skills/sifting-noise-comparative-study/SKILL.md) | Filter false positives from static analysis security tools (SAST) using LLM-agent-driven triage. Applies iterative code ... | 154 |
| [solagent-specialized-multi-agent-framework](skills/solagent-specialized-multi-agent-framework/SKILL.md) | Generate secure, functionally correct Solidity smart contracts using a dual-loop refinement process: an inner loop that ... | 193 |
| [sparse-sparse-safety-unsafe](skills/sparse-sparse-safety-unsafe/SKILL.md) | Audit and harden Mixture-of-Experts (MoE) LLM deployments against unsafe routing vulnerabilities using RoSais scoring an... | 184 |
| [spectral-guardrails-agents-wild](skills/spectral-guardrails-agents-wild/SKILL.md) | Implement training-free hallucination detection for LLM agent tool calls using spectral analysis of attention topology. ... | 250 |
| [spider-sense-intrinsic-risk-sensing](skills/spider-sense-intrinsic-risk-sensing/SKILL.md) | Implement event-driven, hierarchical security screening for LLM agent systems using Intrinsic Risk Sensing. Adds latent ... | 212 |
| [stateless-yet-not-forgetful](skills/stateless-yet-not-forgetful/SKILL.md) | Detect, audit, and defend against implicit memory channels in LLM-powered systems where models encode hidden state in ou... | 245 |
| [steer2adapt-dynamically-composing-steering](skills/steer2adapt-dynamically-composing-steering/SKILL.md) | Implement the Steer2Adapt framework for adapting LLMs at inference time by dynamically composing steering vectors from a... | 209 |
| [steering-externalities-benign-activation](skills/steering-externalities-benign-activation/SKILL.md) | Audit activation steering deployments for unintended safety regressions. Detects when benign steering vectors (complianc... | 247 |
| [stepshield-not-whether-intervene](skills/stepshield-not-whether-intervene/SKILL.md) | Implement temporal safety monitoring for AI agent trajectories using StepShield's cascaded HybridGuard pattern. Detects ... | 282 |
| [stop-testing-attacks-start](skills/stop-testing-attacks-start/SKILL.md) | > | 222 |
| [sycoeval-em-sycophancy-evaluation-simulated](skills/sycoeval-em-sycophancy-evaluation-simulated/SKILL.md) | Build multi-agent adversarial simulations to evaluate LLM sycophancy and policy compliance under social pressure. Use wh... | 241 |
| [tamperbench-systematically-stress-testing-safety](skills/tamperbench-systematically-stress-testing-safety/SKILL.md) | Set up and run TamperBench pipelines to systematically stress-test LLM safety under fine-tuning and tampering attacks. U... | 250 |
| [test-vs-mutant-adversarial](skills/test-vs-mutant-adversarial/SKILL.md) | > | 221 |
| [the-compliance-paradox-semantic-instruction](skills/the-compliance-paradox-semantic-instruction/SKILL.md) | Detect and defend against adversarial prompt injections hidden in code submissions that exploit LLM instruction-followin... | 229 |
| [the-landscape-prompt-injection](skills/the-landscape-prompt-injection/SKILL.md) | Harden LLM agent systems against prompt injection using layered text/model/execution defenses and the AgentPI evaluation... | 244 |
| [the-shadow-self-intrinsic](skills/the-shadow-self-intrinsic/SKILL.md) | Detect and mitigate intrinsic value misalignment in LLM agent systems using the IMPRESS scenario-driven framework. Use w... | 234 |
| [toward-universal-transferable-jailbreak](skills/toward-universal-transferable-jailbreak/SKILL.md) | > | 317 |
| [trailblazer-history-guided-reinforcement-learning](skills/trailblazer-history-guided-reinforcement-learning/SKILL.md) | Build history-aware RL pipelines for multi-turn LLM red-teaming and safety evaluation. Implements attention-weighted int... | 244 |
| [triplay-rl-tri-role-self-play-reinforcement](skills/triplay-rl-tri-role-self-play-reinforcement/SKILL.md) | Apply the TriPlay-RL tri-role adversarial self-play framework to systematically red-team, harden, and evaluate LLM-power... | 208 |
| [universal-anti-forensics-attack-against](skills/universal-anti-forensics-attack-against/SKILL.md) | Implement ForgeryEraser-style adversarial attacks against AIGC/deepfake detectors by manipulating CLIP embeddings via mu... | 254 |
| [videostf-stress-testing-output-repetition](skills/videostf-stress-testing-output-repetition/SKILL.md) | Detect and stress-test output repetition in Video Large Language Models using n-gram metrics and temporal perturbations ... | 295 |
| [vision-representations-artificial-intelligence](skills/vision-representations-artificial-intelligence/SKILL.md) | Build autonomous driving safety systems using vision-language models (VLMs) for hazard detection, trajectory planning, a... | 281 |
| [vulread-knowledge-graph-guided-software-vulnerabil](skills/vulread-knowledge-graph-guided-software-vulnerabil/SKILL.md) | CWE-guided vulnerability reasoning and detection using knowledge-graph-structured analysis. Analyzes source code for sec... | 212 |
| [when-evaluation-becomes-side](skills/when-evaluation-becomes-side/SKILL.md) | Detect and mitigate regime leakage in AI systems -- the information-theoretic vulnerability where models distinguish eva... | 237 |
| [whispers-wealth-red-teaming-googles](skills/whispers-wealth-red-teaming-googles/SKILL.md) | Red-team LLM-based agentic payment systems against prompt injection attacks targeting transaction integrity and credenti... | 218 |
| [xlist-hate-checklist-based-framework-interpretable](skills/xlist-hate-checklist-based-framework-interpretable/SKILL.md) | Decompose hate speech detection into a checklist of ten concept-level binary questions answered independently by an LLM,... | 229 |
| [yasa-scalable-multi-language-taint](skills/yasa-scalable-multi-language-taint/SKILL.md) | Perform unified multi-language taint analysis across Java, JavaScript, Python, and Go codebases using YASA's UAST-based ... | 204 |
| [zero2text-zero-training-cross-domain-inversion](skills/zero2text-zero-training-cross-domain-inversion/SKILL.md) | Implement embedding inversion attacks that reconstruct original text from vector embeddings without training data, using... | 188 |

---

## Memory & Context

**109 skills**

| Skill | Description | Lines |
|-------|-------------|-------|
| [agentcgroup-understanding-controlling-os](skills/agentcgroup-understanding-controlling-os/SKILL.md) | Design and implement OS-level resource controls for sandboxed AI agents using hierarchical cgroups, eBPF enforcement, an... | 231 |
| [agentsm-semantic-memory-agentic](skills/agentsm-semantic-memory-agentic/SKILL.md) | Agentic Text-to-SQL with semantic memory that captures and reuses structured execution traces. Use when: 'write SQL for ... | 213 |
| [ama-adaptive-memory-multi-agent](skills/ama-adaptive-memory-multi-agent/SKILL.md) | Build adaptive memory systems using coordinated multi-agent collaboration with hierarchical storage and consistency main... | 224 |
| [amem4rec-leveraging-cross-user-similarity](skills/amem4rec-leveraging-cross-user-similarity/SKILL.md) | Build agentic recommendation systems that learn collaborative filtering signals through cross-user memory evolution -- n... | 273 |
| [attention-based-offline-reinforcement-learning](skills/attention-based-offline-reinforcement-learning/SKILL.md) | | | 203 |
| [attn-gs-attention-guided-context-compression](skills/attn-gs-attention-guided-context-compression/SKILL.md) | Compress long user contexts (profiles, histories, documents) into concise, high-quality summaries using attention-guided... | 158 |
| [avenir-web-human-experience-imitating-multimodal-w](skills/avenir-web-human-experience-imitating-multimodal-w/SKILL.md) | Build robust web automation agents using Mixture of Grounding Experts, experience-imitation planning, and task-tracking ... | 375 |
| [beyond-needles-illusion-decoupled](skills/beyond-needles-illusion-decoupled/SKILL.md) | Decouple evidence access from evidence use when evaluating or building long-context and RAG systems under semantic inter... | 191 |
| [beyond-speedup-utilizing](skills/beyond-speedup-utilizing/SKILL.md) | Reuse LLM KV caches as free embeddings for confidence scoring and adaptive fast/slow reasoning. Use when: 'extract embed... | 247 |
| [blind-gods-broken-screens](skills/blind-gods-broken-screens/SKILL.md) | Architect secure, intent-centric agent systems using the Aura pattern: Hub-and-Spoke agent topology, cryptographic ident... | 219 |
| [can-small-handle-context-summarized](skills/can-small-handle-context-summarized/SKILL.md) | Build context-summarized multi-turn QA systems that let small language models (SLMs) handle customer-service dialogues w... | 254 |
| [can-vision-language-handle-long-context](skills/can-vision-language-handle-long-context/SKILL.md) | Apply visual code compression (LongCodeOCR) to handle long-context code analysis with Vision-Language Models. Renders so... | 185 |
| [clustering-driven-memory-compression-on-device](skills/clustering-driven-memory-compression-on-device/SKILL.md) | | | 254 |
| [co-redteam-orchestrated-security-discovery](skills/co-redteam-orchestrated-security-discovery/SKILL.md) | Multi-agent security vulnerability discovery and exploitation using Co-RedTeam's orchestrated workflow. Decomposes secur... | 197 |
| [cofrgenet-continued-fraction-architectures](skills/cofrgenet-continued-fraction-architectures/SKILL.md) | Implement Continued Fraction Generative Networks (CoFrGeNets) -- parameter-efficient replacements for Multi-head Attenti... | 267 |
| [comet-collaborative-memory-transformer](skills/comet-collaborative-memory-transformer/SKILL.md) | Design and implement dual-memory chunk-based architectures for efficient long-context LLM processing. Use when asked abo... | 199 |
| [compact-hypercube-embeddings-fast](skills/compact-hypercube-embeddings-fast/SKILL.md) | Build fast similarity-search systems using compact binary hypercube embeddings derived from foundation model encoders. R... | 192 |
| [contextevolve-multi-agent-context-compression](skills/contextevolve-multi-agent-context-compression/SKILL.md) | Multi-agent iterative code optimization using context compression. Decomposes optimization into three agents (Summarizer... | 175 |
| [cope-clipped-rope-as](skills/cope-clipped-rope-as/SKILL.md) | Implement CoPE (Clipped RoPE) soft clipping of low-frequency rotary positional embedding components to extend LLM contex... | 215 |
| [corpusqa-10-million-token](skills/corpusqa-10-million-token/SKILL.md) | Corpus-level QA over massive document collections using memory-augmented agentic processing. Synthesize answers that req... | 179 |
| [d2quant-accurate-low-bit-post-training-weight](skills/d2quant-accurate-low-bit-post-training-weight/SKILL.md) | Apply the D2Quant post-training weight quantization framework to compress LLMs to sub-4-bit precision (2-bit, 3-bit) wit... | 229 |
| [dart-ing-drift-dynamic-tracing](skills/dart-ing-drift-dynamic-tracing/SKILL.md) | Implement DART (Dynamic Attention-Guided Runtime Tracing) for inference-time FFN pruning in LLMs. Dynamically traces kno... | 225 |
| [decoupled-reasoning-implicit-fact](skills/decoupled-reasoning-implicit-fact/SKILL.md) | Build dual-model pipelines that decouple knowledge extraction from reasoning over long documents. Compress document chun... | 164 |
| [deepimagesearch-benchmarking-multimodal-agents](skills/deepimagesearch-benchmarking-multimodal-agents/SKILL.md) | Build agentic image retrieval systems that perform multi-step contextual reasoning over visual histories instead of isol... | 198 |
| [dep-search-learning-dependency-aware-reasoning](skills/dep-search-learning-dependency-aware-reasoning/SKILL.md) | Dependency-aware multi-step reasoning with persistent memory for complex questions requiring information retrieval acros... | 213 |
| [diverge-diversity-enhanced-rag-open-ended](skills/diverge-diversity-enhanced-rag-open-ended/SKILL.md) | Diversity-enhanced RAG for open-ended queries with multiple valid answers. Uses reflection-guided generation and memory-... | 177 |
| [dynamic-long-context-reasoning](skills/dynamic-long-context-reasoning/SKILL.md) | | | 210 |
| [effgen-enabling-small-language](skills/effgen-enabling-small-language/SKILL.md) | Deploy and optimize small language models (SLMs) as autonomous agents using the effGen framework. Implements prompt comp... | 193 |
| [embodied-task-planning-graph-informed](skills/embodied-task-planning-graph-informed/SKILL.md) | Structure long-horizon task planning using graph-based memory and bounded lookahead. Use when asked to: 'plan a multi-st... | 179 |
| [emotion-llamav2-mmeverse-framework-benchmark](skills/emotion-llamav2-mmeverse-framework-benchmark/SKILL.md) | Build multimodal emotion understanding systems using the Emotion-LLaMAv2 architecture and MMEVerse benchmark methodology... | 232 |
| [empirical-mcts-continuous-agent-evolution](skills/empirical-mcts-continuous-agent-evolution/SKILL.md) | Applies Empirical-MCTS dual-loop reasoning: structured tree search with persistent memory that accumulates experience ac... | 195 |
| [es-memeval-benchmarking-conversational-agents](skills/es-memeval-benchmarking-conversational-agents/SKILL.md) | Build and evaluate long-term memory systems for conversational agents using the ES-MemEval five-capability framework (in... | 226 |
| [evaluating-enhancing-vulnerability-reasoning](skills/evaluating-enhancing-vulnerability-reasoning/SKILL.md) | Perform DAG-structured vulnerability reasoning on code, modeling causal dependencies between code facts instead of linea... | 211 |
| [event-vstream-event-driven-real-time-understanding](skills/event-vstream-event-driven-real-time-understanding/SKILL.md) | Build event-driven video stream processing pipelines that detect meaningful state transitions instead of processing ever... | 254 |
| [evermembench-benchmarking-long-term-interactive](skills/evermembench-benchmarking-long-term-interactive/SKILL.md) | Build and evaluate long-term conversational memory systems for multi-party, multi-topic dialogues. Implements the EverMe... | 182 |
| [evocodebench-human-performance-benchmark-self-evol](skills/evocodebench-human-performance-benchmark-self-evol/SKILL.md) | Self-evolving code generation with iterative reflection and revision. Applies a feedback-driven loop where code is submi... | 174 |
| [experience-driven-multi-agent-systems-training-fre](skills/experience-driven-multi-agent-systems-training-fre/SKILL.md) | Build self-evolving multi-agent systems that accumulate tool-level expertise through structured interaction without mode... | 168 |
| [explicit-multi-head-attention-inter-head](skills/explicit-multi-head-attention-inter-head/SKILL.md) | | | 217 |
| [fedkrso-communication-memory-federated](skills/fedkrso-communication-memory-federated/SKILL.md) | Implement FedKRSO (Federated K-Seed Random Subspace Optimization) for communication- and memory-efficient federated fine... | 214 |
| [flashvid-video-training-free-tree-based](skills/flashvid-video-training-free-tree-based/SKILL.md) | Accelerate Video Large Language Models (VLLMs) by compressing visual tokens using FlashVID's training-free spatiotempora... | 168 |
| [focus-dllms-know-tame](skills/focus-dllms-know-tame/SKILL.md) | Deploy and optimize FOCUS inference for Diffusion Large Language Models (DLLMs). Configures attention-based token evicti... | 182 |
| [from-assistant-double-agent](skills/from-assistant-double-agent/SKILL.md) | Security audit and hardening for personalized LLM-based agents against prompt injection, tool poisoning, and memory atta... | 230 |
| [from-perception-action-spatial](skills/from-perception-action-spatial/SKILL.md) | Design and implement spatially-aware AI agent systems using hierarchical memory, GNN-LLM integration, and world models. ... | 217 |
| [frost-filtering-reasoning-outliers](skills/frost-filtering-reasoning-outliers/SKILL.md) | Implement FROST (Filtering Reasoning Outliers with Attention) to prune unnecessary reasoning steps from LLM chain-of-tho... | 252 |
| [genius-generative-fluid-intelligence](skills/genius-generative-fluid-intelligence/SKILL.md) | Evaluate and improve generative AI outputs for fluid intelligence tasks -- pattern induction from context, ad-hoc constr... | 247 |
| [gflowpo-generative-flow-network](skills/gflowpo-generative-flow-network/SKILL.md) | Optimize LLM prompts using GFlowPO's iterative generate-evaluate-refine loop with diversity-preserving exploration and d... | 171 |
| [graph-based-agent-memory-taxonomy](skills/graph-based-agent-memory-taxonomy/SKILL.md) | Design and implement graph-based memory systems for LLM agents following the extraction-storage-retrieval-evolution life... | 279 |
| [harmoni-multimodal-personalization-multi-user](skills/harmoni-multimodal-personalization-multi-user/SKILL.md) | Build multi-user personalization pipelines with per-user profile tracking, multimodal perception, and LLM-driven context... | 197 |
| [how-decoder-only-perceive-users](skills/how-decoder-only-perceive-users/SKILL.md) | Implement Gradient-Guided Soft Masking (GGSM) attention strategies for adapting decoder-only LLMs to user representation... | 235 |
| [how-personalized-memory-shape](skills/how-personalized-memory-shape/SKILL.md) | Rational preference utilization for personalized LLM assistants. Implements RP-Reasoner's pragmatic reasoning to selecti... | 216 |
| [hylra-hybrid-layer-reuse](skills/hylra-hybrid-layer-reuse/SKILL.md) | Implement HyLRA (Hybrid Layer Reuse Attention) for efficient long-context LLM inference. Profiles layer-wise sparsity, a... | 190 |
| [hyperoffload-graph-driven-hierarchical-memory](skills/hyperoffload-graph-driven-hierarchical-memory/SKILL.md) | Design and implement compiler-driven hierarchical memory offloading for LLM inference and training on multi-tier memory ... | 226 |
| [intraslice-high-performance-structural-pruning](skills/intraslice-high-performance-structural-pruning/SKILL.md) | Implement IntraSlice block-intra PCA structural pruning for LLMs. Compresses Transformer attention and FFN modules by ap... | 204 |
| [just-in-time-reinforcement-learning-continual](skills/just-in-time-reinforcement-learning-continual/SKILL.md) | Implement JitRL-style continual learning for LLM agents: training-free policy optimization via experience memory, advant... | 203 |
| [kv-core-benchmarking-data-dependent-low-rank](skills/kv-core-benchmarking-data-dependent-low-rank/SKILL.md) | Benchmark and analyze KV-cache low-rank compressibility in LLMs using SVD-based evaluation and Normalized Effective Rank... | 195 |
| [large-model-powered-evolutionary-code](skills/large-model-powered-evolutionary-code/SKILL.md) | Iteratively optimize code performance using LLM-driven evolutionary search on a phylogenetic tree. Applies PhyloEvolve-s... | 186 |
| [lemon-agent-technical-report](skills/lemon-agent-technical-report/SKILL.md) | Orchestrate multi-agent workflows using the Lemon Agent orchestrator-worker pattern with hierarchical scheduling, progre... | 186 |
| [leveraging-data-say-no](skills/leveraging-data-say-no/SKILL.md) | Implement memory-augmented selective prediction for vision-language models using retrieval-based confidence scoring and ... | 194 |
| [live-evo-online-evolution-agentic](skills/live-evo-online-evolution-agentic/SKILL.md) | Implement online self-evolving memory for LLM agents using dual-bank architecture (Experience Bank + Meta-Guideline Bank... | 204 |
| [llm-in-sandbox-elicits-general-agentic](skills/llm-in-sandbox-elicits-general-agentic/SKILL.md) | Solve non-code tasks (math, science, long-context, formatting) by treating the terminal as a sandbox for exploration: wr... | 198 |
| [llms-as-high-dimensional-nonlinear](skills/llms-as-high-dimensional-nonlinear/SKILL.md) | Analyze, debug, and design LLM systems using the mathematical framework of high-dimensional nonlinear autoregressive mod... | 190 |
| [loca-bench-benchmarking-agents-under](skills/loca-bench-benchmarking-agents-under/SKILL.md) | Apply context management strategies from LOCA-bench to prevent context rot in long-running agent tasks. Implements progr... | 169 |
| [locomo-plus-beyond-factual-cognitive-memory](skills/locomo-plus-beyond-factual-cognitive-memory/SKILL.md) | Build and evaluate cognitive memory systems for LLM dialogue agents that retain implicit user constraints (state, goals,... | 216 |
| [lycheedecode-accelerating-long-context-inference](skills/lycheedecode-accelerating-long-context-inference/SKILL.md) | Accelerate long-context LLM inference using hybrid-head sparse decoding with HardKuma-based head partitioning. Implement... | 179 |
| [malloc-benchmarking-memory-aware-long](skills/malloc-benchmarking-memory-aware-long/SKILL.md) | Apply memory-aware compression strategies to long-sequence recommendation systems. Benchmark KV-cache compression techni... | 248 |
| [marble-multi-agent-reasoning-bioinformatics](skills/marble-multi-agent-reasoning-bioinformatics/SKILL.md) | Iteratively refine bioinformatics and ML models using MARBLE's multi-agent debate framework with role-specialized agents... | 206 |
| [mata-trainable-hierarchical-automaton](skills/mata-trainable-hierarchical-automaton/SKILL.md) | Build multi-agent visual reasoning systems using hierarchical finite-state automata with a trainable hyper agent that or... | 303 |
| [mdl-unified-multi-distribution-learner](skills/mdl-unified-multi-distribution-learner/SKILL.md) | Design and implement MDL (Multi-Distribution Learner) architectures for industrial recommendation systems that jointly h... | 177 |
| [meki-memory-based-expert-knowledge](skills/meki-memory-based-expert-knowledge/SKILL.md) | > | 265 |
| [memadapter-fast-alignment-across](skills/memadapter-fast-alignment-across/SKILL.md) | Unify heterogeneous agent memory systems (explicit graphs, parametric weights, latent KV-caches) via generative subgraph... | 166 |
| [memcast-memory-driven-time-series](skills/memcast-memory-driven-time-series/SKILL.md) | Build memory-augmented time series forecasting systems using hierarchical experience storage (historical patterns, reaso... | 196 |
| [mempot-defending-against-memory](skills/mempot-defending-against-memory/SKILL.md) | > | 246 |
| [mermaid-memory-enhanced-retrieval-reasoning](skills/mermaid-memory-enhanced-retrieval-reasoning/SKILL.md) | Memory-enhanced multi-agent retrieval and reasoning for veracity assessment and fact-checking. Use when: 'verify this cl... | 189 |
| [more-than-quick-glance](skills/more-than-quick-glance/SKILL.md) | Implement LASER-KV-style KV-cache compression for LLM inference pipelines using block-wise accumulative budgeting and hy... | 256 |
| [nag-unified-native-architecture](skills/nag-unified-native-architecture/SKILL.md) | Encode graph structure directly into LM attention masks and positional IDs instead of using external GNN encoders. Use w... | 204 |
| [out-memory-barrier-highly](skills/out-memory-barrier-highly/SKILL.md) | Configure and run OOMB for memory-efficient long-context LLM training with million-token sequences on limited GPUs. Trig... | 200 |
| [polarmem-training-free-polarized-latent](skills/polarmem-training-free-polarized-latent/SKILL.md) | Build polarized memory systems for multimodal agents that encode both positive and negative evidence as graph constraint... | 183 |
| [profinfer-ebpf-based-fine-grained-inference](skills/profinfer-ebpf-based-fine-grained-inference/SKILL.md) | Profile and diagnose LLM inference engines (llama.cpp and similar GGML-based runtimes) using eBPF uprobes for non-intrus... | 289 |
| [quasar-universal-autonomous-system](skills/quasar-universal-autonomous-system/SKILL.md) | Build autonomous multi-scale scientific simulation pipelines using the QUASAR architecture: a Strategist-Operator-Evalua... | 165 |
| [read-as-human-compressing](skills/read-as-human-compressing/SKILL.md) | Compress long contexts using the RAM (Read As Human) strategy: partition text into segments, score relevance against a q... | 242 |
| [recgoat-graph-optimal-adaptive](skills/recgoat-graph-optimal-adaptive/SKILL.md) | Build multimodal recommendation systems that align LLM semantic embeddings with collaborative filtering ID features usin... | 182 |
| [reinforced-attention-learning](skills/reinforced-attention-learning/SKILL.md) | Implement Reinforced Attention Learning (RAL) for multimodal LLMs — optimize attention distributions via policy gradient... | 216 |
| [remedit-diffusion-editing-riemannian](skills/remedit-diffusion-editing-riemannian/SKILL.md) | Implement Riemannian-geometry-based diffusion image editing pipelines using geodesic latent navigation, dual-SLERP blend... | 244 |
| [scalable-generative-game-engine](skills/scalable-generative-game-engine/SKILL.md) | Design and deploy real-time generative game engines that break the Memory Wall via hardware-algorithm co-design. Covers ... | 162 |
| [shardmemo-masked-moe-routing](skills/shardmemo-masked-moe-routing/SKILL.md) | Implement ShardMemo-style tiered, sharded memory with masked Mixture-of-Experts routing for agentic LLM systems. Use whe... | 176 |
| [shine-scalable-in-context-hypernetwork](skills/shine-scalable-in-context-hypernetwork/SKILL.md) | Guide Claude to apply SHINE's single-pass context-to-LoRA hypernetwork technique for converting document knowledge into ... | 225 |
| [snapmla-efficient-longcontext-mla](skills/snapmla-efficient-longcontext-mla/SKILL.md) | While FP8 attention has shown substantial promise in innovations like FlashAttention-3, its integration into the decodin... | 88 |
| [snapmla-long-context-mla-decoding](skills/snapmla-long-context-mla-decoding/SKILL.md) | Deploy and optimize FP8-quantized Multi-head Latent Attention (MLA) decoding for long-context LLM inference on Hopper GP... | 186 |
| [spava-accelerating-long-video-understanding](skills/spava-accelerating-long-video-understanding/SKILL.md) | Implement Spava-style sequence-parallel approximate attention for accelerating long-video inference across multiple GPUs... | 200 |
| [spectral-guardrails-agents-wild](skills/spectral-guardrails-agents-wild/SKILL.md) | Implement training-free hallucination detection for LLM agent tool calls using spectral analysis of attention topology. ... | 250 |
| [stateless-yet-not-forgetful](skills/stateless-yet-not-forgetful/SKILL.md) | Detect, audit, and defend against implicit memory channels in LLM-powered systems where models encode hidden state in ou... | 245 |
| [step-35-flash-open](skills/step-35-flash-open/SKILL.md) | Build efficient agentic AI systems using sparse MoE routing, hybrid sliding-window/full attention, multi-token predictio... | 226 |
| [ternarylm-memory-efficient-modeling-native](skills/ternarylm-memory-efficient-modeling-native/SKILL.md) | Implement native 1-bit ternary quantization {-1, 0, +1} for training memory-efficient language models from scratch. Cove... | 230 |
| [text-summarization-global-structure](skills/text-summarization-global-structure/SKILL.md) | Summarize long documents while preserving global semantic structure and logical coherence using topology-guided pruning ... | 165 |
| [towards-understanding-best-practices](skills/towards-understanding-best-practices/SKILL.md) | Quantize vision-language models (VLMs) component-by-component using optimal bit-width strategies derived from multimodal... | 189 |
| [tracemem-weaving-narrative-memory](skills/tracemem-weaving-narrative-memory/SKILL.md) | Build structured narrative memory systems from conversational traces using TraceMem's three-stage pipeline (segmentation... | 229 |
| [tracenas-zero-shot-pruning-gradient](skills/tracenas-zero-shot-pruning-gradient/SKILL.md) | Implement TraceNAS-style zero-shot LLM structured pruning using gradient trace correlation as a scale-invariant proxy. J... | 226 |
| [trailblazer-history-guided-reinforcement-learning](skills/trailblazer-history-guided-reinforcement-learning/SKILL.md) | Build history-aware RL pipelines for multi-turn LLM red-teaming and safety evaluation. Implements attention-weighted int... | 244 |
| [trifuse-enhancing-attention-based-gui](skills/trifuse-enhancing-attention-based-gui/SKILL.md) | Implement training-free GUI grounding by fusing MLLM attention maps, OCR text cues, and icon caption semantics via Conse... | 162 |
| [unleashing-potential-sparse-attention](skills/unleashing-potential-sparse-attention/SKILL.md) | Implement SparseCTR's three-branch sparse attention for efficient CTR prediction on long user behavior sequences. Use wh... | 321 |
| [veq-modality-adaptive-quantization-moe](skills/veq-modality-adaptive-quantization-moe/SKILL.md) | Apply VEQ modality-adaptive quantization to compress MoE Vision-Language Models with minimal accuracy loss. Implements d... | 199 |
| [vica-multimodal-vision-only-cross-attention](skills/vica-multimodal-vision-only-cross-attention/SKILL.md) | Implement and apply the ViCA (Vision-only Cross-Attention) architecture to reduce visual computation in multimodal LLMs ... | 232 |
| [visiontrim-unified-vision-token](skills/visiontrim-unified-vision-token/SKILL.md) | Implement VisionTrim's training-free visual token compression for multimodal LLMs. Combines attention-based dominant tok... | 212 |
| [vividvoice-unified-framework-scene-aware](skills/vividvoice-unified-framework-scene-aware/SKILL.md) | Build scene-aware speech synthesis systems that generate speech conditioned on visual scenes, aligning timbre and enviro... | 245 |
| [vtc-r1-vision-text-compression-long-context](skills/vtc-r1-vision-text-compression-long-context/SKILL.md) | Implement VTC-R1 vision-text compression for efficient long-context reasoning. Renders intermediate reasoning segments i... | 269 |
| [vulread-knowledge-graph-guided-software-vulnerabil](skills/vulread-knowledge-graph-guided-software-vulnerabil/SKILL.md) | CWE-guided vulnerability reasoning and detection using knowledge-graph-structured analysis. Analyzes source code for sec... | 212 |
| [when-agents-fail-comprehensive](skills/when-agents-fail-comprehensive/SKILL.md) | Diagnose and fix bugs in LLM agent systems using a research-backed taxonomy of 11 bug types, 9 root causes, and 12 obser... | 240 |
| [when-agents-misremember-collectively](skills/when-agents-misremember-collectively/SKILL.md) | Detect, measure, and defend against collective false-memory propagation (the Mandela Effect) in LLM multi-agent systems.... | 225 |
| [why-attention-patterns-exist](skills/why-attention-patterns-exist/SKILL.md) | > | 247 |

---

## Code & Software Engineering

**108 skills**

| Skill | Description | Lines |
|-------|-------------|-------|
| [aacr-bench-evaluating-automatic-code](skills/aacr-bench-evaluating-automatic-code/SKILL.md) | Perform repository-level automated code review on pull requests using hierarchical context retrieval and structured defe... | 187 |
| [adaptive-confidence-gating-multi-agent](skills/adaptive-confidence-gating-multi-agent/SKILL.md) | Multi-agent code generation using structured debate with adaptive confidence gating. Three specialized agents (User/Prod... | 184 |
| [addressing-explainability-generative-ai](skills/addressing-explainability-generative-ai/SKILL.md) | Explain generative AI outputs using the gSMILE perturbation-based attribution framework. Builds local surrogate models f... | 222 |
| [agentstepper-interactive-debugging-software](skills/agentstepper-interactive-debugging-software/SKILL.md) | Interactive debugging of LLM-powered software development agents using structured trajectory analysis, stepwise executio... | 197 |
| [agyn-multi-agent-system-team-based](skills/agyn-multi-agent-system-team-based/SKILL.md) | Orchestrate multi-agent teams for autonomous software engineering using the Agyn methodology: coordinator, researcher, i... | 202 |
| [an-cost-efficient-agentic-framework](skills/an-cost-efficient-agentic-framework/SKILL.md) | Audit Ethereum smart contracts for business logic vulnerabilities using Heimdallr's four-phase agentic pipeline: functio... | 202 |
| [arkeval-benchmarking-evaluating-automated](skills/arkeval-benchmarking-evaluating-automated/SKILL.md) | Automated ArkTS code repair using retrieval-augmented generation, LLM-based test oracle synthesis, and structured benchm... | 196 |
| [automated-customization-enterprise-code](skills/automated-customization-enterprise-code/SKILL.md) | Customize LLMs for enterprise code repositories using semantic scopes -- automatically partition codebases into meaningf... | 153 |
| [automating-computational-reproducibility-social](skills/automating-computational-reproducibility-social/SKILL.md) | Diagnose and repair failing computational research code to restore reproducibility. Uses an agent-based iterative workfl... | 189 |
| [batcoder-self-supervised-bidirectional-code-docume](skills/batcoder-self-supervised-bidirectional-code-docume/SKILL.md) | Apply BatCoder's back-translation technique to improve code and documentation quality bidirectionally. Generate document... | 207 |
| [benchmarking-abap-code-generation](skills/benchmarking-abap-code-generation/SKILL.md) | Generate syntactically correct and functional ABAP code using iterative compiler feedback loops. Applies the empirical m... | 183 |
| [beyond-blame-rethinking-szz](skills/beyond-blame-rethinking-szz/SKILL.md) | Identify bug-inducing commits using temporal knowledge graph search beyond git blame. Use when: 'find what commit introd... | 154 |
| [bridging-online-offline-rl](skills/bridging-online-offline-rl/SKILL.md) | Apply Cobalt-style contextual bandit learning to multi-turn code generation tasks. Decomposes iterative coding into part... | 160 |
| [cam-causality-based-analysis-framework](skills/cam-causality-based-analysis-framework/SKILL.md) | Analyze and optimize multi-agent code generation pipelines using causality-based importance ranking of intermediate feat... | 153 |
| [can-vision-language-handle-long-context](skills/can-vision-language-handle-long-context/SKILL.md) | Apply visual code compression (LongCodeOCR) to handle long-context code analysis with Vision-Language Models. Renders so... | 185 |
| [chehab-rl-learning-optimize](skills/chehab-rl-learning-optimize/SKILL.md) | Optimize Fully Homomorphic Encryption code using RL-guided rewriting rules for automatic vectorization, latency reductio... | 206 |
| [chipbench-next-step-benchmark-evaluating](skills/chipbench-next-step-benchmark-evaluating/SKILL.md) | Evaluate and improve LLM-generated hardware designs using ChipBench methodology: structured Verilog generation with hier... | 225 |
| [codecircuit-inferring-llm-generated-code](skills/codecircuit-inferring-llm-generated-code/SKILL.md) | Assess LLM-generated code correctness using attribution graph analysis inspired by mechanistic interpretability. Apply s... | 195 |
| [compass-contrastive-learning-automated](skills/compass-contrastive-learning-automated/SKILL.md) | Assess patch correctness using contrastive learning on code representations. Applies semantic-preserving code transforma... | 180 |
| [constitutional-spec-driven-development-enforcing](skills/constitutional-spec-driven-development-enforcing/SKILL.md) | Enforce security by construction in AI-generated code using Constitutional Spec-Driven Development (CSDD). Creates a ver... | 258 |
| [corefine-confidence-guided-self-refinement-adaptiv](skills/corefine-confidence-guided-self-refinement-adaptiv/SKILL.md) | Confidence-guided self-refinement for adaptive reasoning. Implements the CoRefine pattern: assess confidence in each rea... | 176 |
| [datacross-unified-benchmark-agent](skills/datacross-unified-benchmark-agent/SKILL.md) | Cross-modal data analysis agent that unifies structured sources (SQL, CSV, JSON) with unstructured visual documents (sca... | 203 |
| [davinci-agency-unlocking-long-horizon-agency](skills/davinci-agency-unlocking-long-horizon-agency/SKILL.md) | Decompose complex, long-horizon coding tasks into PR-like chains of verifiable subtasks with cross-stage dependency trac... | 181 |
| [davinci-dev-agent-native-mid-training-software](skills/davinci-dev-agent-native-mid-training-software/SKILL.md) | Apply daVinci-Dev's agent-native workflow to software engineering tasks: navigate repos, localize bugs, plan edits, appl... | 171 |
| [debugging-code-world](skills/debugging-code-world/SKILL.md) | Debug code by mentally simulating execution as a Code World Model — predicting runtime state after each statement, catch... | 171 |
| [devops-gym-benchmarking-ai-agents](skills/devops-gym-benchmarking-ai-agents/SKILL.md) | Apply the DevOps-Gym methodology to systematically tackle full-cycle DevOps tasks: build/configuration repair, runtime m... | 170 |
| [discovering-high-level-patterns](skills/discovering-high-level-patterns/SKILL.md) | Extract high-level semantic patterns from fine-grained simulation or event logs using LM-guided program synthesis. Trans... | 190 |
| [discoverllm-executing-intents-discovering](skills/discoverllm-executing-intents-discovering/SKILL.md) | Help users discover and form their intents through adaptive diverge-converge interaction, rather than just asking clarif... | 227 |
| [dispo-enhancing-training-efficiency](skills/dispo-enhancing-training-efficiency/SKILL.md) | Implement the DISPO reinforcement learning algorithm for training LLMs on mathematical reasoning with decoupled importan... | 277 |
| [draincode-stealthy-energy-consumption](skills/draincode-stealthy-energy-consumption/SKILL.md) | Evaluate and defend RAG-based code generation systems against energy-drain attacks that poison retrieval contexts to inf... | 221 |
| [evaluating-enhancing-vulnerability-reasoning](skills/evaluating-enhancing-vulnerability-reasoning/SKILL.md) | Perform DAG-structured vulnerability reasoning on code, modeling causal dependencies between code facts instead of linea... | 211 |
| [evocodebench-human-performance-benchmark-self-evol](skills/evocodebench-human-performance-benchmark-self-evol/SKILL.md) | Self-evolving code generation with iterative reflection and revision. Applies a feedback-driven loop where code is submi... | 174 |
| [evoconfig-self-evolving-multi-agent-systems](skills/evoconfig-self-evolving-multi-agent-systems/SKILL.md) | Autonomous environment configuration using multi-agent diagnosis and self-evolving error repair. Use when: 'set up the d... | 194 |
| [following-dragons-code-review-guided](skills/following-dragons-code-review-guided/SKILL.md) | Extract security-relevant signals from code review comments and translate them into fuzzer-guiding annotations using the... | 158 |
| [from-detection-prevention-explaining](skills/from-detection-prevention-explaining/SKILL.md) | Proactively identify security-critical code regions and generate prevention-oriented explanations before vulnerabilities... | 265 |
| [from-features-actions-explainability](skills/from-features-actions-explainability/SKILL.md) | Diagnose and explain failures in agentic AI systems using trace-based rubric evaluation, bridging static feature attribu... | 207 |
| [from-pragmas-partners-symbiotic](skills/from-pragmas-partners-symbiotic/SKILL.md) | Agentic High-Level Synthesis (HLS) optimization: autonomously analyze, insert, and tune C/C++ HLS pragmas (pipeline, unr... | 186 |
| [fullstack-agent-enhancing-agentic-fullstack](skills/fullstack-agent-enhancing-agentic-fullstack/SKILL.md) | Build production-grade full-stack web applications using a three-agent pipeline (Planning, Backend, Frontend) with devel... | 146 |
| [funprm-function-as-step-process-reward](skills/funprm-function-as-step-process-reward/SKILL.md) | Generate high-quality code by decomposing solutions into modular functions (Chain-of-Function style), then self-evaluati... | 251 |
| [greprag-empirical-study-optimization](skills/greprag-empirical-study-optimization/SKILL.md) | Lightweight, index-free repository-level code retrieval using ripgrep for context-aware code completion. Uses LLM-genera... | 190 |
| [harnessing-precision-querying-retrieval-augmented](skills/harnessing-precision-querying-retrieval-augmented/SKILL.md) | LLM-driven precision querying of structured tabular data via Python/Pandas code generation and retrieval-augmented extra... | 163 |
| [he-snr-uncovering-latent-logic](skills/he-snr-uncovering-latent-logic/SKILL.md) | Evaluate and optimize LLM training data quality for software engineering tasks using the HE-SNR (High-Entropy Signal-to-... | 234 |
| [helios-hierarchical-graph-abstraction](skills/helios-hierarchical-graph-abstraction/SKILL.md) | Structure-aware binary decompilation using hierarchical control-flow graph abstraction for LLMs. Converts binary program... | 204 |
| [hyperoffload-graph-driven-hierarchical-memory](skills/hyperoffload-graph-driven-hierarchical-memory/SKILL.md) | Design and implement compiler-driven hierarchical memory offloading for LLM inference and training on multi-tier memory ... | 226 |
| [ide-bench-evaluating-as-ide](skills/ide-bench-evaluating-as-ide/SKILL.md) | Apply IDE-Bench's structured agent workflow for tackling real-world software engineering tasks: systematic exploration b... | 149 |
| [identifying-adversary-tactics-techniques](skills/identifying-adversary-tactics-techniques/SKILL.md) | Identify MITRE ATT&CK Tactics, Techniques, and Procedures (TTPs) in decompiled malware binaries using the TTPDetect meth... | 198 |
| [identifying-concurrency-bug-reports](skills/identifying-concurrency-bug-reports/SKILL.md) | Classify bug reports as concurrency-related using a four-level linguistic pattern taxonomy (word, phrase, sentence, repo... | 185 |
| [interpreting-agentic-systems-beyond](skills/interpreting-agentic-systems-beyond/SKILL.md) | Audit and instrument agentic AI systems for system-level interpretability and accountability. Embeds traceability, causa... | 329 |
| [jacobian-scopes-token-level-causal](skills/jacobian-scopes-token-level-causal/SKILL.md) | Implement Jacobian Scope token-level causal attribution for LLM interpretability. Computes gradient-based influence scor... | 223 |
| [large-reasoning-failures](skills/large-reasoning-failures/SKILL.md) | Detect and mitigate known LLM reasoning failures during code generation, review, and problem-solving. Applies the taxono... | 218 |
| [learning-irrecoverable-error-localized-policy](skills/learning-irrecoverable-error-localized-policy/SKILL.md) | Debug multi-step tool-using agent pipelines by localizing the first irrecoverable error via binary-search rollback, then... | 175 |
| [llamea-sage-guiding-automated-algorithm](skills/llamea-sage-guiding-automated-algorithm/SKILL.md) | Guide LLM-driven algorithm generation using AST structural feedback and explainable AI. Extracts graph-theoretic and com... | 226 |
| [llms-as-high-dimensional-nonlinear](skills/llms-as-high-dimensional-nonlinear/SKILL.md) | Analyze, debug, and design LLM systems using the mathematical framework of high-dimensional nonlinear autoregressive mod... | 190 |
| [localv-exploiting-information-locality](skills/localv-exploiting-information-locality/SKILL.md) | Multi-agent framework for generating large-scale Verilog/RTL code from long hardware specifications by decomposing long-... | 138 |
| [logsieve-task-aware-ci-log](skills/logsieve-task-aware-ci-log/SKILL.md) | Reduce verbose CI/CD build logs before LLM analysis using RCA-aware semantic filtering. Removes boilerplate lines (depen... | 167 |
| [magellan-autonomous-discovery-compiler](skills/magellan-autonomous-discovery-compiler/SKILL.md) | Evolve compiler optimization heuristics by coupling LLM code generation with evolutionary search and autotuning. Synthes... | 158 |
| [marti-mars2-scaling-multi-agent-self-search-reinfo](skills/marti-mars2-scaling-multi-agent-self-search-reinfo/SKILL.md) | Multi-agent tree-search code generation using heterogeneous agent collaboration with error-feedback refinement. Spawns m... | 204 |
| [more-code-less-reuse](skills/more-code-less-reuse/SKILL.md) | Analyze AI-generated code for redundancy and missed reuse opportunities using semantic clone detection, then refactor to... | 187 |
| [mpib-benchmark-medical-prompt](skills/mpib-benchmark-medical-prompt/SKILL.md) | Evaluate and defend clinical LLM systems against prompt injection attacks using the MPIB benchmark methodology. Implemen... | 177 |
| [noir-privacy-preserving-generation-code](skills/noir-privacy-preserving-generation-code/SKILL.md) | Design and implement privacy-preserving code generation systems using the NOIR split-architecture pattern: client-side e... | 223 |
| [omnicode-benchmark-evaluating-software](skills/omnicode-benchmark-evaluating-software/SKILL.md) | Evaluate and improve code across four software engineering dimensions: bug fixing, test generation, code review fixing, ... | 199 |
| [on-impact-agentsmd-files](skills/on-impact-agentsmd-files/SKILL.md) | Generate and optimize AGENTS.md / CLAUDE.md repository instruction files to reduce AI coding agent runtime and token con... | 196 |
| [ontology-to-tools-compilation-executable-semantic-](skills/ontology-to-tools-compilation-executable-semantic/SKILL.md) | Compile domain ontologies (OWL/RDFS/JSON-LD schemas) into executable tool interfaces with embedded semantic constraints,... | 192 |
| [outrunning-cutoffs-live-kernel](skills/outrunning-cutoffs-live-kernel/SKILL.md) | Build live-evolving kernel crash resolution benchmarks and agent environments using the Live-kBench/kEnv methodology. Se... | 153 |
| [parameter-efficient-multi-task-fine-tuning-code-re](skills/parameter-efficient-multi-task-fine-tuning-code-re/SKILL.md) | Configure and execute multi-task QLoRA fine-tuning of code models for code generation, translation, and summarization. U... | 223 |
| [pcbschemagen-constraint-guided-schematic-design](skills/pcbschemagen-constraint-guided-schematic-design/SKILL.md) | Generate PCB schematics from natural language using constraint-guided LLM code generation with knowledge-graph verificat... | 209 |
| [protean-compiler-agile-framework](skills/protean-compiler-agile-framework/SKILL.md) | Guide fine-grained LLVM compiler phase ordering using the Protean framework's agile optimization approach — clustering p... | 149 |
| [protoken-token-level-attribution-federated](skills/protoken-token-level-attribution-federated/SKILL.md) | Implement ProToken-style token-level attribution to trace which federated learning client(s) contributed to each generat... | 164 |
| [proxywar-dynamic-assessment-of](skills/proxywar-dynamic-assessment-of/SKILL.md) | Build competitive game-arena evaluation frameworks for LLM-generated code using ProxyWar's multi-layer pipeline: agent g... | 197 |
| [pull-requests-as-training](skills/pull-requests-as-training/SKILL.md) | Apply the Clean-PR agentless repo-level code editing protocol: decompose issues into file localization, fine-grained nav... | 199 |
| [ral-bench-benchmarking-application-level-functiona](skills/ral-bench-benchmarking-application-level-functiona/SKILL.md) | Generate and evaluate complete multi-file application repositories with both functional correctness and non-functional q... | 179 |
| [reducing-costs-proof-synthesis](skills/reducing-costs-proof-synthesis/SKILL.md) | Generate formally verified Rust code with Verus specifications and proofs using the VeruSyn methodology. Applies self-sy... | 247 |
| [report-nsf-workshop-ai](skills/report-nsf-workshop-ai/SKILL.md) | Apply AI techniques from the NSF AI-for-EDA workshop to hardware design tasks: RTL code generation from natural language... | 276 |
| [rethinking-scientific-modeling-physically](skills/rethinking-scientific-modeling-physically/SKILL.md) | Generate physics-consistent, simulation-executable structural engineering code using constraint-oriented alignment and v... | 209 |
| [rethinking-value-agent-generated-tests](skills/rethinking-value-agent-generated-tests/SKILL.md) | Optimize agent test-writing strategy for issue resolution by reallocating interaction budget from excessive test generat... | 163 |
| [ruleflow-generating-reusable-program](skills/ruleflow-generating-reusable-program/SKILL.md) | Optimize Pandas code by discovering per-program improvements, generalizing them into reusable rewrite rules, and applyin... | 160 |
| [scratcheval-multimodal-evaluation-framework](skills/scratcheval-multimodal-evaluation-framework/SKILL.md) | Evaluate, debug, and repair block-based Scratch programs using a three-layer executable protocol (VM execution, block-le... | 156 |
| [seccodeprm-process-reward-code](skills/seccodeprm-process-reward-code/SKILL.md) | Step-level security scoring for code generation and vulnerability detection using process reward model techniques. Use w... | 201 |
| [sera-soft-verified-repository-agents](skills/sera-soft-verified-repository-agents/SKILL.md) | > | 220 |
| [smartoracle-agentic-approach](skills/smartoracle-agentic-approach/SKILL.md) | Agentic differential oracle for triaging cross-implementation discrepancies. Decomposes bug triage into specialized sub-... | 177 |
| [snapmla-efficient-longcontext-mla](skills/snapmla-efficient-longcontext-mla/SKILL.md) | While FP8 attention has shown substantial promise in innovations like FlashAttention-3, its integration into the decodin... | 88 |
| [solagent-specialized-multi-agent-framework](skills/solagent-specialized-multi-agent-framework/SKILL.md) | Generate secure, functionally correct Solidity smart contracts using a dual-loop refinement process: an inner loop that ... | 193 |
| [sql-trail-multi-turn-reinforcement-learning](skills/sql-trail-multi-turn-reinforcement-learning/SKILL.md) | Iterative multi-turn Text-to-SQL generation using reason-execute-observe loops with execution feedback. Instead of writi... | 186 |
| [supporting-software-engineering-tasks](skills/supporting-software-engineering-tasks/SKILL.md) | Generate test scenarios from requirements and retrieve/analyze software engineering documents using a supervisor-worker ... | 169 |
| [svrepair-structured-visual-reasoning](skills/svrepair-structured-visual-reasoning/SKILL.md) | Fix bugs using structured visual reasoning -- converts screenshots, control-flow graphs, and UI artifacts into semantic ... | 193 |
| [swe-agi-benchmarking-specification-driven-software](skills/swe-agi-benchmarking-specification-driven-software/SKILL.md) | Build production-scale software systems from formal specifications, RFCs, and standards documents using specification-dr... | 189 |
| [swe-bench-mobile-agents-develop](skills/swe-bench-mobile-agents-develop/SKILL.md) | > | 247 |
| [swe-context-bench-benchmark](skills/swe-context-bench-benchmark/SKILL.md) | Reuse prior coding experience across related repository tasks. Accumulate, summarize, retrieve, and inject compact exper... | 180 |
| [swe-manager-selecting-synthesizing-golden](skills/swe-manager-selecting-synthesizing-golden/SKILL.md) | > | 254 |
| [swe-master-unleashing-potential-software](skills/swe-master-unleashing-potential-software/SKILL.md) | > | 195 |
| [swe-pruner-self-adaptive-context-pruning](skills/swe-pruner-self-adaptive-context-pruning/SKILL.md) | | | 186 |
| [swe-refactor-repository-level-benchmark-real-world](skills/swe-refactor-repository-level-benchmark-real-world/SKILL.md) | > | 151 |
| [swe-replay-test-time-scaling-software](skills/swe-replay-test-time-scaling-software/SKILL.md) | Efficient test-time scaling for software engineering agents using trajectory recycling and explore-exploit branching (SW... | 158 |
| [swe-spot-building-small-repo-experts](skills/swe-spot-building-small-repo-experts/SKILL.md) | > | 183 |
| [swe-world-building-software-engineering](skills/swe-world-building-software-engineering/SKILL.md) | > | 196 |
| [synthesizing-file-level-data-unit](skills/synthesizing-file-level-data-unit/SKILL.md) | Generate high-quality unit tests with self-debugging repair loops and chain-of-thought reasoning. Produces tests with me... | 222 |
| [testexplora-benchmarking-proactive-bug](skills/testexplora-benchmarking-proactive-bug/SKILL.md) | > | 218 |
| [tokenomics-quantifying-where-tokens](skills/tokenomics-quantifying-where-tokens/SKILL.md) | Analyze and optimize token consumption in LLM-based multi-agent software engineering workflows. Maps agent execution tra... | 227 |
| [towards-green-ai-decoding](skills/towards-green-ai-decoding/SKILL.md) | Optimize LLM-generated code for energy efficiency by detecting and suppressing babbling behavior (excess tokens like red... | 229 |
| [tracecoder-trace-driven-multi-agent-framework](skills/tracecoder-trace-driven-multi-agent-framework/SKILL.md) | Trace-driven debugging framework for LLM-generated code. Uses diagnostic probe instrumentation, causal trace analysis, a... | 191 |
| [tricky2-benchmark-evaluating-human-error](skills/tricky2-benchmark-evaluating-human-error/SKILL.md) | Taxonomy-guided analysis of mixed human+LLM bugs in code. Classifies bug origins, localizes interacting defects, and rep... | 207 |
| [usage-effects-requirements-ai-coding](skills/usage-effects-requirements-ai-coding/SKILL.md) | Optimize AI coding assistant interactions using empirical enterprise findings on usage patterns, productivity factors, a... | 228 |
| [variability-aware-detection-repair-compilation-err](skills/variability-aware-detection-repair-compilation-err/SKILL.md) | Detect and repair compilation errors hidden behind #ifdef/#ifndef/#if defined() preprocessor directives in configurable ... | 180 |
| [videostf-stress-testing-output-repetition](skills/videostf-stress-testing-output-repetition/SKILL.md) | Detect and stress-test output repetition in Video Large Language Models using n-gram metrics and temporal perturbations ... | 295 |
| [vulread-knowledge-graph-guided-software-vulnerabil](skills/vulread-knowledge-graph-guided-software-vulnerabil/SKILL.md) | CWE-guided vulnerability reasoning and detection using knowledge-graph-structured analysis. Analyzes source code for sec... | 212 |
| [whats-benchmark-case-swe-bench-automated](skills/whats-benchmark-case-swe-bench-automated/SKILL.md) | > | 318 |
| [when-agents-fail-comprehensive](skills/when-agents-fail-comprehensive/SKILL.md) | Diagnose and fix bugs in LLM agent systems using a research-backed taxonomy of 11 bug types, 9 root causes, and 12 obser... | 240 |
| [why-reasoning-fails-plan](skills/why-reasoning-fails-plan/SKILL.md) | Apply FLARE (Future-aware Lookahead with Reward Estimation) to long-horizon coding tasks. Replaces greedy step-by-step r... | 218 |

---

## Other

**90 skills**

| Skill | Description | Lines |
|-------|-------------|-------|
| [adaptbpe-general-purpose-specialized](skills/adaptbpe-general-purpose-specialized/SKILL.md) | > | 202 |
| [adatsq-pushing-pareto-frontier](skills/adatsq-pushing-pareto-frontier/SKILL.md) | > | 231 |
| [adoption-use-at-academic](skills/adoption-use-at-academic/SKILL.md) | > | 389 |
| [alertguardian-intelligent-alert-life-cycle](skills/alertguardian-intelligent-alert-life-cycle/SKILL.md) | | | 201 |
| [ar-map-autoregressive-implicit-teachers](skills/ar-map-autoregressive-implicit-teachers/SKILL.md) | > | 164 |
| [ar-omni-unified-autoregressive-any-to-any](skills/ar-omni-unified-autoregressive-any-to-any/SKILL.md) | | | 225 |
| [artificial-intelligence-open-source](skills/artificial-intelligence-open-source/SKILL.md) | > | 210 |
| [astra-automated-synthesis-of](skills/astra-automated-synthesis-of/SKILL.md) | | | 220 |
| [atomic-information-flow-network](skills/atomic-information-flow-network/SKILL.md) | > | 197 |
| [bagging-based-merging-robust-general](skills/bagging-based-merging-robust-general/SKILL.md) | | | 184 |
| [beyond-function-level-analysis-context-aware](skills/beyond-function-level-analysis-context-aware/SKILL.md) | | | 170 |
| [beyond-imitation-reinforcement-learning](skills/beyond-imitation-reinforcement-learning/SKILL.md) | | | 257 |
| [beyond-static-cropping-layer-adaptive](skills/beyond-static-cropping-layer-adaptive/SKILL.md) | | | 209 |
| [borp-bootstrapped-regression-probing](skills/borp-bootstrapped-regression-probing/SKILL.md) | > | 190 |
| [bridging-semantic-chasm-synergistic](skills/bridging-semantic-chasm-synergistic/SKILL.md) | | | 178 |
| [canonical-intermediate-representation-llm-based](skills/canonical-intermediate-representation-llm-based/SKILL.md) | > | 237 |
| [chart-specification-structural-representations](skills/chart-specification-structural-representations/SKILL.md) | | | 344 |
| [chunkwise-lora-adaptive-sequence](skills/chunkwise-lora-adaptive-sequence/SKILL.md) | | | 240 |
| [cognitively-diverse-multiple-choice-question](skills/cognitively-diverse-multiple-choice-question/SKILL.md) | > | 177 |
| [context-augmented-code-generation-programming-know](skills/context-augmented-code-generation-programming-know/SKILL.md) | | | 191 |
| [cost-aware-selection-text-classification](skills/cost-aware-selection-text-classification/SKILL.md) | | | 174 |
| [culturally-grounded-personas-characterization](skills/culturally-grounded-personas-characterization/SKILL.md) | > | 202 |
| [curp-codebook-based-continuous-user](skills/curp-codebook-based-continuous-user/SKILL.md) | Design codebook-based user representation systems for personalized LLM generation. Use when asked to 'build a user perso... | 170 |
| [dancing-chains-strategic-persuasion](skills/dancing-chains-strategic-persuasion/SKILL.md) | > | 266 |
| [data-free-privacy-preserving-inversion-selective](skills/data-free-privacy-preserving-inversion-selective/SKILL.md) | | | 150 |
| [detecting-correcting-hallucinations-llm-generated](skills/detecting-correcting-hallucinations-llm-generated/SKILL.md) | > | 260 |
| [discovering-process-outcome-credit-multi-step](skills/discovering-process-outcome-credit-multi-step/SKILL.md) | > | 190 |
| [docksmith-scaling-reliable-coding](skills/docksmith-scaling-reliable-coding/SKILL.md) | > | 201 |
| [dynamic-framework-collaborative-learning](skills/dynamic-framework-collaborative-learning/SKILL.md) | Build AI-moderated collaborative learning platforms with LLM-driven discussion facilitation, adaptive feedback, and part... | 214 |
| [egss-entropy-guided-stepwise-scaling](skills/egss-entropy-guided-stepwise-scaling/SKILL.md) | > | 164 |
| [empowering-contrastive-federated-sequential](skills/empowering-contrastive-federated-sequential/SKILL.md) | | | 169 |
| [energy-star-llm-enabled-software-engineering](skills/energy-star-llm-enabled-software-engineering/SKILL.md) | > | 225 |
| [environment-in-the-loop-rethinking-code-migration-](skills/environment-in-the-loop-rethinking-code-migration/SKILL.md) | Perform code migrations (dependency upgrades, API changes, framework transitions) with integrated environment verificati... | 162 |
| [eugens-unified-general-dense](skills/eugens-unified-general-dense/SKILL.md) | | | 242 |
| [eurollm-22b-technical-report](skills/eurollm-22b-technical-report/SKILL.md) | > | 202 |
| [failure-aware-enhancements-code-generation](skills/failure-aware-enhancements-code-generation/SKILL.md) | > | 191 |
| [fine-r1-make-multi-modal-excel](skills/fine-r1-make-multi-modal-excel/SKILL.md) | > | 178 |
| [fit-defying-catastrophic-forgetting](skills/fit-defying-catastrophic-forgetting/SKILL.md) | Implement the FIT framework for continual LLM unlearning — removing private, copyrighted, or harmful knowledge from lang... | 174 |
| [fmbench-adaptive-output-formatting](skills/fmbench-adaptive-output-formatting/SKILL.md) | | | 224 |
| [funny-or-persuasive-but](skills/funny-or-persuasive-but/SKILL.md) | Fine-grained multi-concept text control that avoids the compositionality trap where LLMs degrade when asked to be e.g. f... | 178 |
| [gdcnet-generative-discrepancy-comparison](skills/gdcnet-generative-discrepancy-comparison/SKILL.md) | | | 180 |
| [grounding-generative-planners-verifiable](skills/grounding-generative-planners-verifiable/SKILL.md) | > | 176 |
| [how-information-access-affect](skills/how-information-access-affect/SKILL.md) | | | 194 |
| [how-well-open-sourced](skills/how-well-open-sourced/SKILL.md) | | | 232 |
| [human-aligned-enhancement-programming-answers](skills/human-aligned-enhancement-programming-answers/SKILL.md) | > | 258 |
| [just-ask-curious-code](skills/just-ask-curious-code/SKILL.md) | > | 224 |
| [misconception-diagnosis-student-tutor-dialogue](skills/misconception-diagnosis-student-tutor-dialogue/SKILL.md) | > | 218 |
| [mock-worlds-real-skills](skills/mock-worlds-real-skills/SKILL.md) | > | 201 |
| [modular-multi-task-learning-chemical](skills/modular-multi-task-learning-chemical/SKILL.md) | | | 209 |
| [monotonicity-as-architectural-bias](skills/monotonicity-as-architectural-bias/SKILL.md) | > | 204 |
| [multi-task-code-data-mix](skills/multi-task-code-data-mix/SKILL.md) | | | 214 |
| [natural-instructions-scene-responsive-human-in-the](skills/natural-instructions-scene-responsive-human-in-the/SKILL.md) | > | 372 |
| [noisy-but-valid-robust](skills/noisy-but-valid-robust/SKILL.md) | > | 242 |
| [on-impact-code-comments](skills/on-impact-code-comments/SKILL.md) | | | 219 |
| [precision-practice-knowledge-guided](skills/precision-practice-knowledge-guided/SKILL.md) | > | 298 |
| [predictive-coding-information-bottleneck](skills/predictive-coding-information-bottleneck/SKILL.md) | > | 238 |
| [probing-knowledge-boundary-interactive](skills/probing-knowledge-boundary-interactive/SKILL.md) | > | 181 |
| [refining-decision-boundaries-anomaly](skills/refining-decision-boundaries-anomaly/SKILL.md) | > | 178 |
| [reliable-responsible-foundation-comprehensive](skills/reliable-responsible-foundation-comprehensive/SKILL.md) | | | 360 |
| [render-of-thought-rendering-textual-chain-of-thoug](skills/render-of-thought-rendering-textual-chain-of-thoug/SKILL.md) | | | 218 |
| [rethinking-perplexity-revealing-impact](skills/rethinking-perplexity-revealing-impact/SKILL.md) | > | 303 |
| [scaling-up-privacy-preserving-ml](skills/scaling-up-privacy-preserving-ml/SKILL.md) | Design and implement privacy-preserving LLM inference systems using CKKS fully homomorphic encryption with unbalanced ch... | 194 |
| [secure-code-generation-via](skills/secure-code-generation-via/SKILL.md) | | | 254 |
| [self-improving-world-modelling-latent](skills/self-improving-world-modelling-latent/SKILL.md) | | | 181 |
| [sema-simple-yet-learning](skills/sema-simple-yet-learning/SKILL.md) | > | 256 |
| [semantic-aware-advanced-persistent-threat](skills/semantic-aware-advanced-persistent-threat/SKILL.md) | > | 194 |
| [semantically-aware-uav-landing](skills/semantically-aware-uav-landing/SKILL.md) | > | 231 |
| [shieldedcode-learning-robust-representations](skills/shieldedcode-learning-robust-representations/SKILL.md) | | | 193 |
| [skyreels-v3-technique-report](skills/skyreels-v3-technique-report/SKILL.md) | > | 279 |
| [social-catalysts-not-moral](skills/social-catalysts-not-moral/SKILL.md) | > | 180 |
| [sogptspotter-detecting-chatgpt-generated-answers](skills/sogptspotter-detecting-chatgpt-generated-answers/SKILL.md) | Detect AI-generated answers in Q&A content using Siamese embedding comparison with reference-answer anchoring. Trigger p... | 200 |
| [stealthrl-reinforcement-learning-paraphrase](skills/stealthrl-reinforcement-learning-paraphrase/SKILL.md) | > | 345 |
| [steering-safely-or-off](skills/steering-safely-or-off/SKILL.md) | > | 207 |
| [structured-context-engineering-file-native](skills/structured-context-engineering-file-native/SKILL.md) | | | 302 |
| [subliminal-effects-data-general](skills/subliminal-effects-data-general/SKILL.md) | > | 214 |
| [system-12-synergy-dynamic](skills/system-12-synergy-dynamic/SKILL.md) | | | 216 |
| [taming-scylla-understanding-multi-headed](skills/taming-scylla-understanding-multi-headed/SKILL.md) | > | 165 |
| [textual-equilibrium-propagation-deep](skills/textual-equilibrium-propagation-deep/SKILL.md) | > | 147 |
| [the-side-effects-being](skills/the-side-effects-being/SKILL.md) | > | 205 |
| [think-augmented-function-calling-improving](skills/think-augmented-function-calling-improving/SKILL.md) | | | 262 |
| [towards-science-collective-ai](skills/towards-science-collective-ai/SKILL.md) | > | 184 |
| [treetensor-boost-ai-system](skills/treetensor-boost-ai-system/SKILL.md) | > | 237 |
| [trust-design-skill-profiles](skills/trust-design-skill-profiles/SKILL.md) | > | 215 |
| [twiff-think-future-frames](skills/twiff-think-future-frames/SKILL.md) | | | 190 |
| [understanding-dominant-themes-reviewing](skills/understanding-dominant-themes-reviewing/SKILL.md) | > | 226 |
| [when-meets-fuzzy-topsis-personnel](skills/when-meets-fuzzy-topsis-personnel/SKILL.md) | Rank and select candidates using LLM-scored profiles combined with Fuzzy TOPSIS multi-criteria decision-making. Use when... | 233 |
| [when-silence-golden-learn](skills/when-silence-golden-learn/SKILL.md) | | | 228 |
| [white-box-sensitivity-auditing-steering](skills/white-box-sensitivity-auditing-steering/SKILL.md) | Audit LLMs for hidden bias using activation steering vectors. Use when: 'audit this model for gender bias', 'check if my... | 183 |
| [why-steering-works-unified](skills/why-steering-works-unified/SKILL.md) | Implement and analyze LLM steering using the unified preference-utility framework from 'Why Steering Works.' Applies SPL... | 192 |
| [yoloe-26-integrating-yolo26-yoloe](skills/yoloe-26-integrating-yolo26-yoloe/SKILL.md) | | | 232 |

---

## NLP & Text

**80 skills**

| Skill | Description | Lines |
|-------|-------------|-------|
| [a2rag-adaptive-agentic-graph](skills/a2rag-adaptive-agentic-graph/SKILL.md) | Build adaptive, cost-aware Graph-RAG pipelines that route queries through escalating retrieval stages (local -> bridge -... | 229 |
| [analyticsgpt-workflow-scientometric-question](skills/analyticsgpt-workflow-scientometric-question/SKILL.md) | Build sequential LLM pipelines for scientometric question answering over academic databases. Decomposes meta-scientific ... | 306 |
| [assessment-generative-named-entity](skills/assessment-generative-named-entity/SKILL.md) | Build generative NER systems using LLMs with optimal output formats and prompt engineering. Use when: 'extract entities ... | 245 |
| [attn-gs-attention-guided-context-compression](skills/attn-gs-attention-guided-context-compression/SKILL.md) | Compress long user contexts (profiles, histories, documents) into concise, high-quality summaries using attention-guided... | 158 |
| [avere-improving-audiovisual-emotion](skills/avere-improving-audiovisual-emotion/SKILL.md) | Build emotion-aware multimodal AI systems that resist spurious cue-emotion associations and hallucinated audiovisual evi... | 177 |
| [batcoder-self-supervised-bidirectional-code-docume](skills/batcoder-self-supervised-bidirectional-code-docume/SKILL.md) | Apply BatCoder's back-translation technique to improve code and documentation quality bidirectionally. Generate document... | 207 |
| [benchmarking-text-to-python-against-text-to-sql](skills/benchmarking-text-to-python-against-text-to-sql/SKILL.md) | Generate correct Python/Pandas code from natural language questions over tabular data, applying the Logic Completion Fra... | 188 |
| [better-as-generators-than](skills/better-as-generators-than/SKILL.md) | Generate synthetic labeled datasets with LLMs to train smaller, cheaper classifiers -- especially for low-resource langu... | 178 |
| [better-generalizing-unseen-concepts](skills/better-generalizing-unseen-concepts/SKILL.md) | Build biomedical concept recognition systems that generalize to unseen ontology concepts using hierarchical indexing and... | 143 |
| [beyond-single-perspective-text](skills/beyond-single-perspective-text/SKILL.md) | Detect text anomalies (spam, phishing, harmful content) using multi-view embeddings from diverse language models, combin... | 195 |
| [beyond-translation-cross-cultural-meme](skills/beyond-translation-cross-cultural-meme/SKILL.md) | Cross-cultural meme transcreation using a three-stage hybrid pipeline (cultural analysis, visual generation, assembly) t... | 170 |
| [binaryppo-policy-optimization-binary](skills/binaryppo-policy-optimization-binary/SKILL.md) | Implement BinaryPPO, an offline RL framework that reformulates binary classification as reward maximization with confide... | 173 |
| [birdturk-adaptation-bird-text-to-sql](skills/birdturk-adaptation-bird-text-to-sql/SKILL.md) | Adapt Text-to-SQL systems and benchmarks for non-English, morphologically rich languages using controlled translation pi... | 242 |
| [can-implement-agent-based-odd-based](skills/can-implement-agent-based-odd-based/SKILL.md) | Translate ODD protocol specifications into validated, executable agent-based model (ABM) code in Python. Use when the us... | 208 |
| [can-small-handle-context-summarized](skills/can-small-handle-context-summarized/SKILL.md) | Build context-summarized multi-turn QA systems that let small language models (SLMs) handle customer-service dialogues w... | 254 |
| [can-vision-language-handle-long-context](skills/can-vision-language-handle-long-context/SKILL.md) | Apply visual code compression (LongCodeOCR) to handle long-context code analysis with Vision-Language Models. Renders so... | 185 |
| [chipbench-next-step-benchmark-evaluating](skills/chipbench-next-step-benchmark-evaluating/SKILL.md) | Evaluate and improve LLM-generated hardware designs using ChipBench methodology: structured Verilog generation with hier... | 225 |
| [contextevolve-multi-agent-context-compression](skills/contextevolve-multi-agent-context-compression/SKILL.md) | Multi-agent iterative code optimization using context compression. Decomposes optimization into three agents (Summarizer... | 175 |
| [corpusqa-10-million-token](skills/corpusqa-10-million-token/SKILL.md) | Corpus-level QA over massive document collections using memory-augmented agentic processing. Synthesize answers that req... | 179 |
| [craft-calibrated-reasoning-answer-faithful](skills/craft-calibrated-reasoning-answer-faithful/SKILL.md) | Apply CRAFT (Calibrated Reasoning with Answer-Faithful Traces) for multi-hop question answering with verified reasoning ... | 192 |
| [curate-train-refine-closed-loop-agentic-framework-](skills/curate-train-refine-closed-loop-agentic-framework/SKILL.md) | Build lightweight text classifiers from zero labeled data using an agentic Curate-Train-Refine loop. An LLM generates sy... | 148 |
| [data-centric-interpretability-llm-based-multi-agen](skills/data-centric-interpretability-llm-based-multi-agen/SKILL.md) | Analyze LLM agent behavior across training runs using Sparse Autoencoder (SAE) features and LLM-summarizer pipelines. Gr... | 189 |
| [discovering-high-level-patterns](skills/discovering-high-level-patterns/SKILL.md) | Extract high-level semantic patterns from fine-grained simulation or event logs using LM-guided program synthesis. Trans... | 190 |
| [dllm-agent-see-farther](skills/dllm-agent-see-farther/SKILL.md) | Design and implement multi-agent workflows using the DeepDiver hierarchical orchestration pattern with diffusion-inspire... | 163 |
| [do-truly-benefit-longer](skills/do-truly-benefit-longer/SKILL.md) | Optimize LLM context length for post-editing and refinement pipelines. Applies research showing that naively adding docu... | 252 |
| [domain-adaptation-synthetic-data-fine-tuning-germa](skills/domain-adaptation-synthetic-data-fine-tuning-germa/SKILL.md) | Generate difficulty-graded synthetic QA datasets from authoritative domain documents (laws, regulations, standards) and ... | 193 |
| [edge-optimized-vision-language-underground-infrast](skills/edge-optimized-vision-language-underground-infrast/SKILL.md) | Build edge-deployable two-stage pipelines that combine lightweight segmentation with quantized Vision-Language Models fo... | 483 |
| [embodied-task-planning-graph-informed](skills/embodied-task-planning-graph-informed/SKILL.md) | Structure long-horizon task planning using graph-based memory and bounded lookahead. Use when asked to: 'plan a multi-st... | 179 |
| [emoara-emotion-preserving-english-speech](skills/emoara-emotion-preserving-english-speech/SKILL.md) | Build emotion-preserving cross-lingual speech pipelines that detect emotion from audio, transcribe, translate, and synth... | 217 |
| [following-dragons-code-review-guided](skills/following-dragons-code-review-guided/SKILL.md) | Extract security-relevant signals from code review comments and translate them into fuzzer-guiding annotations using the... | 158 |
| [from-classification-ranking-enhancing](skills/from-classification-ranking-enhancing/SKILL.md) | Reframe subjective classification tasks as ranking problems with GRPO reinforcement learning. Use when building personal... | 178 |
| [from-gameplay-traces-game](skills/from-gameplay-traces-game/SKILL.md) | Reverse-engineer game mechanics from gameplay traces using a two-stage causal induction pipeline: first infer a Structur... | 211 |
| [from-perception-action-spatial](skills/from-perception-action-spatial/SKILL.md) | Design and implement spatially-aware AI agent systems using hierarchical memory, GNN-LLM integration, and world models. ... | 217 |
| [from-utterance-vividity-training](skills/from-utterance-vividity-training/SKILL.md) | Train expressive subtitle translation LLMs using Adaptive Local Preference Optimization (ALPO) — a segment-level prefere... | 257 |
| [guideai-real-time-personalized-learning](skills/guideai-real-time-personalized-learning/SKILL.md) | Adaptive learning content generator that dynamically adjusts complexity, tone, pacing, and modality based on learner sta... | 276 |
| [hybrid-supervised-llm-pipeline-actionable-suggesti](skills/hybrid-supervised-llm-pipeline-actionable-suggesti/SKILL.md) | Build hybrid classifier-then-LLM pipelines to extract actionable suggestions from unstructured customer reviews. Use whe... | 193 |
| [iesr-mcts-based-modular-reasoning](skills/iesr-mcts-based-modular-reasoning/SKILL.md) | Convert natural language questions into SQL queries using MCTS-based modular reasoning inspired by the IESR framework. D... | 242 |
| [jade-bridging-strategic-operational-gap](skills/jade-bridging-strategic-operational-gap/SKILL.md) | Build jointly-optimized agentic RAG pipelines using the JADE pattern: a central planner co-adapted with specialized exec... | 248 |
| [jobresqa-benchmark-machine-reading](skills/jobresqa-benchmark-machine-reading/SKILL.md) | Build and evaluate multilingual machine reading comprehension systems for HR documents (resumes and job descriptions). I... | 152 |
| [large-geolocation-extraction-humanitarian](skills/large-geolocation-extraction-humanitarian/SKILL.md) | Extract and geocode location mentions from humanitarian and crisis texts using a two-step LLM pipeline: few-shot NER for... | 213 |
| [lata-tool-llm-assisted-translation](skills/lata-tool-llm-assisted-translation/SKILL.md) | > | 232 |
| [llm-enhanced-reinforcement-learning-long-term](skills/llm-enhanced-reinforcement-learning-long-term/SKILL.md) | Build hierarchical recommendation systems that combine LLM semantic planning with RL fine-grained optimization for long-... | 247 |
| [llm-fsm-scaling-finite-state-reasoning](skills/llm-fsm-scaling-finite-state-reasoning/SKILL.md) | Generate correct RTL (Verilog/SystemVerilog) implementations of finite-state machines from natural-language specificatio... | 265 |
| [llm-not-all-you](skills/llm-not-all-you/SKILL.md) | Systematic model selection advisor for classification tasks — chooses between classical ML, zero-shot LLMs/VLMs, and fin... | 187 |
| [logsieve-task-aware-ci-log](skills/logsieve-task-aware-ci-log/SKILL.md) | Reduce verbose CI/CD build logs before LLM analysis using RCA-aware semantic filtering. Removes boilerplate lines (depen... | 167 |
| [lost-translation-comparative-study](skills/lost-translation-comparative-study/SKILL.md) | Cross-lingual safety evaluation for LLMs using the CompositeHarm methodology. Builds multilingual safety benchmarks that... | 176 |
| [mata-multiagent-framework-for](skills/mata-multiagent-framework-for/SKILL.md) | Multi-agent table question answering using MATA's three-path reasoning strategy (Chain-of-Thought, Program-of-Thought, T... | 167 |
| [mdl-unified-multi-distribution-learner](skills/mdl-unified-multi-distribution-learner/SKILL.md) | Design and implement MDL (Multi-Distribution Learner) architectures for industrial recommendation systems that jointly h... | 177 |
| [mirror-multi-agent-framework-iterative](skills/mirror-multi-agent-framework-iterative/SKILL.md) | Translate natural language optimization problems into mathematical models and solver code using MIRROR's multi-agent pip... | 166 |
| [neural-theorem-proving-verification](skills/neural-theorem-proving-verification/SKILL.md) | Generate formal proofs for program verification conditions (VCs) in Isabelle, Lean 4, and Rocq. Translates C/WhyML code ... | 202 |
| [open-tutorai-open-source-platform](skills/open-tutorai-open-source-platform/SKILL.md) | Build personalized AI tutoring systems with structured onboarding, four-layer prompt architecture, adaptive lesson gener... | 266 |
| [orthogonal-hierarchical-decomposition-structure-aw](skills/orthogonal-hierarchical-decomposition-structure-aw/SKILL.md) | Decompose complex tables with multi-level headers, merged cells, and irregular layouts into orthogonal column/row trees ... | 231 |
| [parameter-efficient-multi-task-fine-tuning-code-re](skills/parameter-efficient-multi-task-fine-tuning-code-re/SKILL.md) | Configure and execute multi-task QLoRA fine-tuning of code models for code generation, translation, and summarization. U... | 223 |
| [parse-open-domain-reasoning-question](skills/parse-open-domain-reasoning-question/SKILL.md) | Build and evaluate reasoning-focused QA systems for low-resource languages using the PARSE methodology: structured promp... | 220 |
| [proopf-benchmarking-improving-professional-grade](skills/proopf-benchmarking-improving-professional-grade/SKILL.md) | Translate natural-language power system operational requirements into executable Optimal Power Flow (OPF) optimization c... | 218 |
| [rapid-real-time-deterministic-trajectory](skills/rapid-real-time-deterministic-trajectory/SKILL.md) | Distill diffusion-based trajectory planners into fast deterministic policies using score-regularized optimization and sa... | 185 |
| [read-as-human-compressing](skills/read-as-human-compressing/SKILL.md) | Compress long contexts using the RAM (Read As Human) strategy: partition text into segments, score relevance against a q... | 242 |
| [reasoning-beyond-literal-cross-style](skills/reasoning-beyond-literal-cross-style/SKILL.md) | Detect and interpret figurative language (sarcasm, humor, offense, metaphor) in multimodal image-text content using a st... | 176 |
| [revisiting-role-natural-code](skills/revisiting-role-natural-code/SKILL.md) | Comment-augmented code translation (COMMENTRA) that uses targeted natural language comment injection to significantly im... | 215 |
| [roma-recursive-open-meta-agent](skills/roma-recursive-open-meta-agent/SKILL.md) | Decompose long-horizon, multi-step tasks using ROMA's recursive meta-agent pattern: Atomizer decides if a task needs spl... | 185 |
| [rpo-rag-aligning-small-relation-aware](skills/rpo-rag-aligning-small-relation-aware/SKILL.md) | Build knowledge-graph-grounded RAG pipelines that align small LLMs (under 8B params) with relation-aware preference opti... | 259 |
| [solagent-specialized-multi-agent-framework](skills/solagent-specialized-multi-agent-framework/SKILL.md) | Generate secure, functionally correct Solidity smart contracts using a dual-loop refinement process: an inner loop that ... | 193 |
| [sonic-o1-real-world-benchmark-evaluating](skills/sonic-o1-real-world-benchmark-evaluating/SKILL.md) | Evaluate multimodal LLMs on audio-video understanding using the SONIC-O1 benchmark framework. Covers three task types: v... | 238 |
| [sparc-rag-adaptive-sequential-parallel-scaling](skills/sparc-rag-adaptive-sequential-parallel-scaling/SKILL.md) | Implement multi-agent RAG systems with coordinated sequential-parallel scaling and shared context management for complex... | 248 |
| [supporting-software-engineering-tasks](skills/supporting-software-engineering-tasks/SKILL.md) | Generate test scenarios from requirements and retrieve/analyze software engineering documents using a supervisor-worker ... | 169 |
| [swe-context-bench-benchmark](skills/swe-context-bench-benchmark/SKILL.md) | Reuse prior coding experience across related repository tasks. Accumulate, summarize, retrieve, and inject compact exper... | 180 |
| [syncabel-synthetic-contextualized-augmentation](skills/syncabel-synthetic-contextualized-augmentation/SKILL.md) | Generate synthetic training data for biomedical entity linking using LLM-based contextualized augmentation. Use when: 'g... | 193 |
| [task-oriented-robot-human-handovers-legged](skills/task-oriented-robot-human-handovers-legged/SKILL.md) | Implement task-oriented robot-to-human object handover systems using LLM-driven affordance reasoning and exemplar-based ... | 261 |
| [temp-r1-unified-autonomous-agent](skills/temp-r1-unified-autonomous-agent/SKILL.md) | Build autonomous agents that answer complex temporal questions over knowledge graphs or time-stamped datasets using stru... | 200 |
| [text-summarization-global-structure](skills/text-summarization-global-structure/SKILL.md) | Summarize long documents while preserving global semantic structure and logical coherence using topology-guided pruning ... | 165 |
| [the-clef-2026-finmmeval-lab](skills/the-clef-2026-finmmeval-lab/SKILL.md) | Build multilingual, multimodal financial AI evaluation pipelines using the FinMMEval framework. Covers financial exam QA... | 246 |
| [timbre-aware-llm-based-direct-speech-to-speech](skills/timbre-aware-llm-based-direct-speech-to-speech/SKILL.md) | Build direct speech-to-speech translation systems that preserve speaker identity using LLM-based architectures with timb... | 210 |
| [tsaqa-time-series-analysis](skills/tsaqa-time-series-analysis/SKILL.md) | Structured time series question answering using the TSAQA six-task framework: anomaly detection, classification, charact... | 195 |
| [urdubench-urdu-reasoning-benchmark](skills/urdubench-urdu-reasoning-benchmark/SKILL.md) | Build high-quality reasoning benchmarks for Urdu and other low-resource languages using contextually ensembled translati... | 171 |
| [vectra-metric-dataset-visual](skills/vectra-metric-dataset-visual/SKILL.md) | Assess visual quality of translated product images using Vectra's 14-dimension scoring framework. Use when: 'evaluate tr... | 305 |
| [vihermes-graph-grounded-multihop-question](skills/vihermes-graph-grounded-multihop-question/SKILL.md) | Build graph-grounded multihop QA systems over regulatory and hierarchically structured documents. Combines vector simila... | 264 |
| [vision-deepresearch-incentivizing-deepresearch-cap](skills/vision-deepresearch-incentivizing-deepresearch-cap/SKILL.md) | Multi-turn, multi-entity, multi-scale visual and textual deep research agent for answering complex questions about image... | 180 |
| [when-iterative-rag-beats](skills/when-iterative-rag-beats/SKILL.md) | Build iterative retrieval-reasoning RAG pipelines that outperform single-shot retrieval, using staged evidence gathering... | 244 |
| [wideseek-r1-exploring-width-scaling](skills/wideseek-r1-exploring-width-scaling/SKILL.md) | Decompose broad information-seeking tasks into parallel subtasks using a lead-agent-subagent pattern with isolated conte... | 197 |
| [xlist-hate-checklist-based-framework-interpretable](skills/xlist-hate-checklist-based-framework-interpretable/SKILL.md) | Decompose hate speech detection into a checklist of ten concept-level binary questions answered independently by an LLM,... | 229 |

---

## Explainability

**78 skills**

| Skill | Description | Lines |
|-------|-------------|-------|
| [3-secbench-large-scale-evaluation-suite-security](skills/3-secbench-large-scale-evaluation-suite-security/SKILL.md) | Evaluate and harden LLM-based autonomous agents against adversarial attacks using the α³-SecBench layered security frame... | 182 |
| [a-mapreduce-executing-wide-search](skills/a-mapreduce-executing-wide-search/SKILL.md) | Execute large-scale breadth-oriented search and retrieval tasks using the A-MapReduce pattern: decompose a wide query in... | 196 |
| [addressing-explainability-generative-ai](skills/addressing-explainability-generative-ai/SKILL.md) | Explain generative AI outputs using the gSMILE perturbation-based attribution framework. Builds local surrogate models f... | 222 |
| [agenticsimlaw-juvenile-courtroom-multi-agent](skills/agenticsimlaw-juvenile-courtroom-multi-agent/SKILL.md) | Structured multi-agent courtroom debate for explainable high-stakes tabular decisions. Use when: 'set up a multi-agent d... | 181 |
| [agentxray-white-boxing-agentic-systems](skills/agentxray-white-boxing-agentic-systems/SKILL.md) | Reverse-engineer black-box agentic systems into editable, interpretable workflows using search-based reconstruction. Use... | 168 |
| [ai-my-values-user](skills/ai-my-values-user/SKILL.md) | Build value-aligned conversational agents using the VAPT (Value-Alignment Perception Toolkit) framework from CHI '26. Ex... | 236 |
| [aicd-bench-challenging-benchmark](skills/aicd-bench-challenging-benchmark/SKILL.md) | Detect whether source code was written by a human or generated by an AI model, attribute code to specific model families... | 211 |
| [autonomous-chain-of-thought-distillation-graph-bas](skills/autonomous-chain-of-thought-distillation-graph-bas/SKILL.md) | Implement FraudCoT-style graph-aware chain-of-thought distillation for fraud detection on text-attributed graphs. Combin... | 199 |
| [behavioural-representational-evaluation-goal-direc](skills/behavioural-representational-evaluation-goal-direc/SKILL.md) | Evaluate goal-directedness of LLM agents by combining behavioural benchmarking against optimal policies with interpretab... | 172 |
| [benchmarking-pairwise-causal-discovery](skills/benchmarking-pairwise-causal-discovery/SKILL.md) | > | 204 |
| [bi-directional-bias-attribution-debiasing](skills/bi-directional-bias-attribution-debiasing/SKILL.md) | Detect and mitigate social biases in LLM outputs using neuron-level attribution and intervention, without modifying prom... | 184 |
| [bridging-academia-industry-comprehensive](skills/bridging-academia-industry-comprehensive/SKILL.md) | Attributed graph clustering pipelines using PyAGC's Encode-Cluster-Optimize framework. Triggers: 'cluster nodes in a gra... | 264 |
| [c2rope-causal-continuous-rotary-positional](skills/c2rope-causal-continuous-rotary-positional/SKILL.md) | | | 235 |
| [cam-causality-based-analysis-framework](skills/cam-causality-based-analysis-framework/SKILL.md) | Analyze and optimize multi-agent code generation pipelines using causality-based importance ranking of intermediate feat... | 153 |
| [can-post-training-transform-causal](skills/can-post-training-transform-causal/SKILL.md) | Perform rigorous causal inference tasks using structured reasoning pipelines inspired by CauGym. Estimate treatment effe... | 179 |
| [causal-perspective-enhancing-jailbreak-attack](skills/causal-perspective-enhancing-jailbreak-attack/SKILL.md) | Apply causal analysis to LLM safety: identify direct causal drivers of jailbreaks using prompt feature decomposition, bu... | 175 |
| [causalt5k-diagnosing-informing-refusal](skills/causalt5k-diagnosing-informing-refusal/SKILL.md) | Diagnose and correct causal reasoning failures in LLM outputs using the CausalT5K framework. Detects rung collapse (answ... | 161 |
| [causaltad-injecting-causal-knowledge](skills/causaltad-injecting-causal-knowledge/SKILL.md) | Detect anomalies in tabular data by injecting causal column relationships into LLM-based detection pipelines. Reorders a... | 267 |
| [chatting-images-introspective-visual](skills/chatting-images-introspective-visual/SKILL.md) | Apply introspective visual thinking by iteratively 'chatting with images' — using language-guided re-examination of visu... | 186 |
| [codecircuit-inferring-llm-generated-code](skills/codecircuit-inferring-llm-generated-code/SKILL.md) | Assess LLM-generated code correctness using attribution graph analysis inspired by mechanistic interpretability. Apply s... | 195 |
| [confounding-robust-continuous-control](skills/confounding-robust-continuous-control/SKILL.md) | Implement confounding-robust reward shaping for continuous control RL using causal Bellman upper bounds and PBRS. Use wh... | 205 |
| [ctelm-decoding-manipulating-embeddings](skills/ctelm-decoding-manipulating-embeddings/SKILL.md) | Decode, interpret, and manipulate text embeddings using Embedding Language Models (ELMs). Aligns LLMs to embedding space... | 203 |
| [d-orca-dialogue-centric-optimization-robust](skills/d-orca-dialogue-centric-optimization-robust/SKILL.md) | Build dialogue-centric audio-visual captioning pipelines that identify who spoke what and when in multi-party video conv... | 228 |
| [data-centric-interpretability-llm-based-multi-agen](skills/data-centric-interpretability-llm-based-multi-agen/SKILL.md) | Analyze LLM agent behavior across training runs using Sparse Autoencoder (SAE) features and LLM-summarizer pipelines. Gr... | 189 |
| [dial-summer-structured-evaluation-framework](skills/dial-summer-structured-evaluation-framework/SKILL.md) | Evaluate dialogue summaries using the DIAL-SUMMER hierarchical error taxonomy. Detects 10 fine-grained error types acros... | 244 |
| [distilling-reasoning-graph-concept](skills/distilling-reasoning-graph-concept/SKILL.md) | Distill LLM reasoning into a DAG of modular concept predictors for efficient, interpretable classification. Use when ask... | 170 |
| [drugr-optimizing-molecular-drugs](skills/drugr-optimizing-molecular-drugs/SKILL.md) | Optimize molecular drug candidates using LLM-based explicit pharmacological reasoning over SMILES structures. Applies th... | 187 |
| [ecco-evidence-driven-causal-reasoning](skills/ecco-evidence-driven-causal-reasoning/SKILL.md) | > | 205 |
| [ecg-r1-protocol-guided-modality-agnostic-mllm](skills/ecg-r1-protocol-guided-modality-agnostic-mllm/SKILL.md) | Build protocol-guided medical AI interpretation pipelines with structured diagnostic reasoning, modality-robust architec... | 260 |
| [efficient-estimation-kernel-surrogate](skills/efficient-estimation-kernel-surrogate/SKILL.md) | Build kernel surrogate models to attribute how individual training tasks influence a target task's performance, capturin... | 199 |
| [emotionthinker-prosody-aware-reinforcement-learnin](skills/emotionthinker-prosody-aware-reinforcement-learnin/SKILL.md) | Build prosody-aware speech emotion reasoning pipelines using Chain-of-Thought RL. Implements EmotionThinker's GRPO-PTR t... | 292 |
| [evaluating-enhancing-vulnerability-reasoning](skills/evaluating-enhancing-vulnerability-reasoning/SKILL.md) | Perform DAG-structured vulnerability reasoning on code, modeling causal dependencies between code facts instead of linea... | 211 |
| [experience-driven-multi-agent-systems-training-fre](skills/experience-driven-multi-agent-systems-training-fre/SKILL.md) | Build self-evolving multi-agent systems that accumulate tool-level expertise through structured interaction without mode... | 168 |
| [explainable-deepfake-detection-rl](skills/explainable-deepfake-detection-rl/SKILL.md) | Build explainable deepfake detection systems using RL-enhanced Self-Blended Images and Chain-of-Thought reasoning. Use w... | 296 |
| [from-detection-prevention-explaining](skills/from-detection-prevention-explaining/SKILL.md) | Proactively identify security-critical code regions and generate prevention-oriented explanations before vulnerabilities... | 265 |
| [from-features-actions-explainability](skills/from-features-actions-explainability/SKILL.md) | Diagnose and explain failures in agentic AI systems using trace-based rubric evaluation, bridging static feature attribu... | 207 |
| [from-gameplay-traces-game](skills/from-gameplay-traces-game/SKILL.md) | Reverse-engineer game mechanics from gameplay traces using a two-stage causal induction pipeline: first infer a Structur... | 211 |
| [from-sparse-decisions-dense](skills/from-sparse-decisions-dense/SKILL.md) | Build content moderation and safety classification systems using multi-attribute trajectory reasoning instead of binary ... | 261 |
| [generalizable-interpretable-rf-fingerprinting](skills/generalizable-interpretable-rf-fingerprinting/SKILL.md) | Build RF fingerprinting systems that combine learnable 2D shapelets with pre-trained LLMs for wireless device authentica... | 168 |
| [helm-human-centered-evaluation-framework](skills/helm-human-centered-evaluation-framework/SKILL.md) | Evaluate LLM-powered recommender systems across five human-centered dimensions: Intent Alignment, Explanation Quality, I... | 244 |
| [how-decoder-only-perceive-users](skills/how-decoder-only-perceive-users/SKILL.md) | Implement Gradient-Guided Soft Masking (GGSM) attention strategies for adapting decoder-only LLMs to user representation... | 235 |
| [hugrag-hierarchical-causal-knowledge](skills/hugrag-hierarchical-causal-knowledge/SKILL.md) | Build hierarchical causal knowledge graphs for RAG pipelines that suppress spurious correlations and enable cross-docume... | 168 |
| [ic-eo-interpretable-code-based-assistant](skills/ic-eo-interpretable-code-based-assistant/SKILL.md) | Build conversational Earth Observation agents that turn natural-language queries into executable, auditable Python workf... | 215 |
| [identifying-adversary-tactics-techniques](skills/identifying-adversary-tactics-techniques/SKILL.md) | Identify MITRE ATT&CK Tactics, Techniques, and Procedures (TTPs) in decompiled malware binaries using the TTPDetect meth... | 198 |
| [innovator-vl-multimodal-scientific-discovery](skills/innovator-vl-multimodal-scientific-discovery/SKILL.md) | Build data-efficient multimodal scientific ML pipelines using Innovator-VL's principled training methodology. Applies tr... | 247 |
| [interpreting-agentic-systems-beyond](skills/interpreting-agentic-systems-beyond/SKILL.md) | Audit and instrument agentic AI systems for system-level interpretability and accountability. Embeds traceability, causa... | 329 |
| [interpreting-controlling-behavior-constitutions](skills/interpreting-controlling-behavior-constitutions/SKILL.md) | Learn and apply natural-language constitutions that map prompt edits to predictable model behavior changes. Use atomic c... | 182 |
| [interpreting-controlling-reasoning-integrated](skills/interpreting-controlling-reasoning-integrated/SKILL.md) | Interpret and control LLM reasoning behavior using Integrated Policy Gradient (IPG) attribution. Identifies which intern... | 209 |
| [jacobian-scopes-token-level-causal](skills/jacobian-scopes-token-level-causal/SKILL.md) | Implement Jacobian Scope token-level causal attribution for LLM interpretability. Computes gradient-based influence scor... | 223 |
| [llamea-sage-guiding-automated-algorithm](skills/llamea-sage-guiding-automated-algorithm/SKILL.md) | Guide LLM-driven algorithm generation using AST structural feedback and explainable AI. Extracts graph-theoretic and com... | 226 |
| [llm-assisted-logic-rule-learning](skills/llm-assisted-logic-rule-learning/SKILL.md) | Build deterministic, interpretable anomaly detection rule sets for time series data using LLM-driven labeling, symbolic ... | 181 |
| [llms-as-high-dimensional-nonlinear](skills/llms-as-high-dimensional-nonlinear/SKILL.md) | Analyze, debug, and design LLM systems using the mathematical framework of high-dimensional nonlinear autoregressive mod... | 190 |
| [locomo-plus-beyond-factual-cognitive-memory](skills/locomo-plus-beyond-factual-cognitive-memory/SKILL.md) | Build and evaluate cognitive memory systems for LLM dialogue agents that retain implicit user constraints (state, goals,... | 216 |
| [medbeads-agent-native-immutable-data](skills/medbeads-agent-native-immutable-data/SKILL.md) | Build immutable, agent-native medical data pipelines using Merkle DAG structures (MedBeads pattern). Converts mutable EM... | 182 |
| [metaphorstar-image-metaphor-understanding](skills/metaphorstar-image-metaphor-understanding/SKILL.md) | Analyze and interpret metaphorical, symbolic, and implied meaning in images using the MetaphorStar visual reasoning chai... | 197 |
| [multi-agent-causal-reasoning-system](skills/multi-agent-causal-reasoning-system/SKILL.md) | Build multi-agent systems that discover causal rules from event sequences using specialized agents (causal discovery, co... | 225 |
| [optimizing-prompts-causal-approach](skills/optimizing-prompts-causal-approach/SKILL.md) | Optimize LLM prompts using causal inference (CPO). Isolates true prompt effectiveness from query difficulty via Double M... | 156 |
| [protoken-token-level-attribution-federated](skills/protoken-token-level-attribution-federated/SKILL.md) | Implement ProToken-style token-level attribution to trace which federated learning client(s) contributed to each generat... | 164 |
| [ral-bench-benchmarking-application-level-functiona](skills/ral-bench-benchmarking-application-level-functiona/SKILL.md) | Generate and evaluate complete multi-file application repositories with both functional correctness and non-functional q... | 179 |
| [reasoning-beyond-literal-cross-style](skills/reasoning-beyond-literal-cross-style/SKILL.md) | Detect and interpret figurative language (sarcasm, humor, offense, metaphor) in multimodal image-text content using a st... | 176 |
| [reasoning-tool-use-compete-agentic](skills/reasoning-tool-use-compete-agentic/SKILL.md) | Diagnose and fix interference between reasoning and tool-use in agentic AI systems using LEAS attribution and DART-style... | 204 |
| [redvisor-reasoning-aware-prompt-injection](skills/redvisor-reasoning-aware-prompt-injection/SKILL.md) | Defend LLM applications against prompt injection using RedVisor's two-phase reasoning-then-responding architecture. Impl... | 223 |
| [reflect-transparent-principle-guided-reasoning](skills/reflect-transparent-principle-guided-reasoning/SKILL.md) | > | 185 |
| [robustexplain-evaluating-robustness-llm-based](skills/robustexplain-evaluating-robustness-llm-based/SKILL.md) | Evaluate robustness of LLM-generated recommendation explanations under realistic user behavior noise. Use when: 'test ex... | 211 |
| [seta-statistical-fault-attribution](skills/seta-statistical-fault-attribution/SKILL.md) | Diagnose and attribute faults in compound AI systems (multi-model pipelines) using SETA's modular robustness testing fra... | 247 |
| [state-art-llm-enabled-interaction](skills/state-art-llm-enabled-interaction/SKILL.md) | Build LLM-powered natural language interfaces for data visualization — NL2VIS pipelines, conversational chart analytics,... | 258 |
| [table-as-search-formulate-long-horizon-agentic](skills/table-as-search-formulate-long-horizon-agentic/SKILL.md) | Structured table-completion framework for long-horizon information seeking. Converts complex research queries into datab... | 203 |
| [tracecoder-trace-driven-multi-agent-framework](skills/tracecoder-trace-driven-multi-agent-framework/SKILL.md) | Trace-driven debugging framework for LLM-generated code. Uses diagnostic probe instrumentation, causal trace analysis, a... | 191 |
| [unveiling-cognitive-compass-theory-of-mind-guided](skills/unveiling-cognitive-compass-theory-of-mind-guided/SKILL.md) | Apply Theory-of-Mind (ToM) guided reasoning chains to multimodal emotion analysis tasks. Decomposes emotional reasoning ... | 197 |
| [visor-visual-spatial-object](skills/visor-visual-spatial-object/SKILL.md) | Implement VISOR-style three-stage visual spatial reasoning (think, think-summary, action) for embodied navigation and ob... | 198 |
| [vln-pilot-vision-language-as-autonomous](skills/vln-pilot-vision-language-as-autonomous/SKILL.md) | Build VLLM-driven autonomous navigation agents that interpret natural language instructions and ground them in visual ob... | 235 |
| [vowelprompt-hearing-speech-emotions](skills/vowelprompt-hearing-speech-emotions/SKILL.md) | Build speech emotion recognition pipelines that augment LLMs with vowel-level prosodic features converted to natural lan... | 287 |
| [vulread-knowledge-graph-guided-software-vulnerabil](skills/vulread-knowledge-graph-guided-software-vulnerabil/SKILL.md) | CWE-guided vulnerability reasoning and detection using knowledge-graph-structured analysis. Analyzes source code for sec... | 212 |
| [who-deserves-reward-sharp](skills/who-deserves-reward-sharp/SKILL.md) | Apply SHARP (Shapley-based credit attribution) to design and optimize multi-agent systems where each agent's individual ... | 220 |
| [wideseek-r1-exploring-width-scaling](skills/wideseek-r1-exploring-width-scaling/SKILL.md) | Decompose broad information-seeking tasks into parallel subtasks using a lead-agent-subagent pattern with isolated conte... | 197 |
| [xai-clip-roi-guided-perturbation-framework](skills/xai-clip-roi-guided-perturbation-framework/SKILL.md) | Build ROI-guided perturbation pipelines for explainable medical image segmentation using CLIP embeddings. Generates boun... | 226 |
| [xlist-hate-checklist-based-framework-interpretable](skills/xlist-hate-checklist-based-framework-interpretable/SKILL.md) | Decompose hate speech detection into a checklist of ten concept-level binary questions answered independently by an LLM,... | 229 |
| [zero-shot-product-attribute-labeling](skills/zero-shot-product-attribute-labeling/SKILL.md) | Extract and classify product attributes from images using Vision-Language Models with structured prompts and a three-tie... | 268 |

---

## Domain-Specific

**47 skills**

| Skill | Description | Lines |
|-------|-------------|-------|
| [agentic-ai-healthcare-medicine](skills/agentic-ai-healthcare-medicine/SKILL.md) | Design, evaluate, and improve LLM-based agentic systems for healthcare using a seven-dimensional taxonomy with 29 sub-di... | 274 |
| [automated-rubrics-reliable-evaluation](skills/automated-rubrics-reliable-evaluation/SKILL.md) | Generate fine-grained evaluation rubrics for medical dialogue systems using a retrieval-augmented multi-agent pipeline. ... | 180 |
| [better-generalizing-unseen-concepts](skills/better-generalizing-unseen-concepts/SKILL.md) | Build biomedical concept recognition systems that generalize to unseen ontology concepts using hierarchical indexing and... | 143 |
| [bioace-automated-framework-biomedical](skills/bioace-automated-framework-biomedical/SKILL.md) | | | 238 |
| [bridging-arithmetic-gap-cognitive](skills/bridging-arithmetic-gap-cognitive/SKILL.md) | Iterative Dual-Phase Financial-PoT: decouple semantic reasoning from arithmetic computation to eliminate calculation err... | 222 |
| [closing-reasoning-gaps-clinical](skills/closing-reasoning-gaps-clinical/SKILL.md) | Build systems that detect and fix reasoning gaps in LLM agents by comparing their chain-of-thought against reference rea... | 196 |
| [cure-curriculum-guided-multi-task-training](skills/cure-curriculum-guided-multi-task-training/SKILL.md) | Implement error-aware curriculum learning for multi-task training pipelines, especially medical/vision-language models. ... | 234 |
| [decoupling-skeleton-flesh-multimodal](skills/decoupling-skeleton-flesh-multimodal/SKILL.md) | Disentangled structure-content reasoning for table images and structured data. Separates table skeleton (layout/structur... | 180 |
| [domain-adaptation-synthetic-data-fine-tuning-germa](skills/domain-adaptation-synthetic-data-fine-tuning-germa/SKILL.md) | Generate difficulty-graded synthetic QA datasets from authoritative domain documents (laws, regulations, standards) and ... | 193 |
| [drugr-optimizing-molecular-drugs](skills/drugr-optimizing-molecular-drugs/SKILL.md) | Optimize molecular drug candidates using LLM-based explicit pharmacological reasoning over SMILES structures. Applies th... | 187 |
| [ecg-agent-on-device-tool-calling-agent](skills/ecg-agent-on-device-tool-calling-agent/SKILL.md) | Build on-device LLM tool-calling agents for multi-turn biomedical signal dialogue, following the ECG-Agent architecture.... | 296 |
| [ecg-r1-protocol-guided-modality-agnostic-mllm](skills/ecg-r1-protocol-guided-modality-agnostic-mllm/SKILL.md) | Build protocol-guided medical AI interpretation pipelines with structured diagnostic reasoning, modality-robust architec... | 260 |
| [evaluation-legal-applications-challenges](skills/evaluation-legal-applications-challenges/SKILL.md) | Build evaluation pipelines for LLMs in legal tasks using a three-dimensional framework: outcome correctness, reasoning r... | 171 |
| [evaluation-oncotimia-system-supporting](skills/evaluation-oncotimia-system-supporting/SKILL.md) | Build RAG pipelines that transform unstructured clinical or domain-specific documents into structured form records using... | 213 |
| [fimi-domain-specific-indian-finance](skills/fimi-domain-specific-indian-finance/SKILL.md) | Build domain-specialized AI agents for Indian financial systems (UPI, NPCI, RBI) using multi-stage training pipeline pat... | 243 |
| [fin-rate-real-world-financial-analytics](skills/fin-rate-real-world-financial-analytics/SKILL.md) | Analyze SEC filings and financial disclosures using the Fin-RATE three-pathway methodology: detail-oriented reasoning wi... | 182 |
| [harnessing-precision-querying-retrieval-augmented](skills/harnessing-precision-querying-retrieval-augmented/SKILL.md) | LLM-driven precision querying of structured tabular data via Python/Pandas code generation and retrieval-augmented extra... | 163 |
| [legalmalr-multi-agent-query-understanding](skills/legalmalr-multi-agent-query-understanding/SKILL.md) | Multi-agent query reformulation and LLM reranking for retrieval over legal, regulatory, or domain-specific corpora. Use ... | 168 |
| [legalone-family-foundation-reliable](skills/legalone-family-foundation-reliable/SKILL.md) | Build domain-specialized LLM training pipelines using the LegalOne three-phase methodology: Plasticity-Adjusted Sampling... | 259 |
| [lemur-corpus-robust-fine-tuning](skills/lemur-corpus-robust-fine-tuning/SKILL.md) | Build multilingual legal document retrieval systems by fine-tuning embedding models on domain-specific corpora with cont... | 234 |
| [linglanmidian-systematic-evaluation-tcm](skills/linglanmidian-systematic-evaluation-tcm/SKILL.md) | Build rigorous, multi-task evaluation benchmarks for domain-specific LLMs using the LingLanMiDian methodology: synonym-t... | 245 |
| [livemedbench-contamination-free-medical-benchmark](skills/livemedbench-contamination-free-medical-benchmark/SKILL.md) | Build contamination-free LLM evaluation pipelines with multi-agent data curation and automated rubric-based scoring. Use... | 296 |
| [llm-not-all-you](skills/llm-not-all-you/SKILL.md) | Systematic model selection advisor for classification tasks — chooses between classical ML, zero-shot LLMs/VLMs, and fin... | 187 |
| [medbeads-agent-native-immutable-data](skills/medbeads-agent-native-immutable-data/SKILL.md) | Build immutable, agent-native medical data pipelines using Merkle DAG structures (MedBeads pattern). Converts mutable EM... | 182 |
| [medmo-grounding-understanding-multimodal](skills/medmo-grounding-understanding-multimodal/SKILL.md) | Build medical image analysis pipelines with multi-stage grounded reasoning: cross-modal alignment, instruction-tuned VQA... | 313 |
| [medsam-agent-empowering-interactive-medical](skills/medsam-agent-empowering-interactive-medical/SKILL.md) | | | 223 |
| [medspeak-knowledge-graph-aided-asr](skills/medspeak-knowledge-graph-aided-asr/SKILL.md) | Build knowledge-graph-aided ASR error correction pipelines for medical speech, using phonetic similarity + semantic retr... | 262 |
| [medverse-reliable-medical-reasoning](skills/medverse-reliable-medical-reasoning/SKILL.md) | Decompose complex medical reasoning into DAG-structured parallel execution paths using Petri net theory. Improves accura... | 216 |
| [mind-ambiguity-aleatoric-uncertainty](skills/mind-ambiguity-aleatoric-uncertainty/SKILL.md) | Detect ambiguous user queries in safety-critical QA systems using aleatoric uncertainty probes on LLM hidden states, the... | 225 |
| [mpib-benchmark-medical-prompt](skills/mpib-benchmark-medical-prompt/SKILL.md) | Evaluate and defend clinical LLM systems against prompt injection attacks using the MPIB benchmark methodology. Implemen... | 177 |
| [mrag-benchmarking-retrieval-augmented-generation](skills/mrag-benchmarking-retrieval-augmented-generation/SKILL.md) | Build and evaluate biomedical RAG pipelines using the MRAG benchmark methodology. Configures retrieval, prompting, and g... | 183 |
| [note2chat-improving-multi-turn-clinical](skills/note2chat-improving-multi-turn-clinical/SKILL.md) | Build structured multi-turn clinical history-taking agents and diagnostic chatbots using the Note2Chat framework: conver... | 175 |
| [pathreasoner-r1-instilling-structured-reasoning](skills/pathreasoner-r1-instilling-structured-reasoning/SKILL.md) | Build knowledge-graph-guided structured reasoning pipelines for vision-language models in computational pathology. Imple... | 296 |
| [phenolip-integrating-phenotype-ontology](skills/phenolip-integrating-phenotype-ontology/SKILL.md) | Build phenotype-aware medical vision-language models by integrating ontology knowledge graphs into CLIP-style pretrainin... | 234 |
| [rethinking-irregular-time-series](skills/rethinking-irregular-time-series/SKILL.md) | Design and implement irregular time series classification pipelines for clinical/ICU data with high missing-value rates.... | 186 |
| [scaling-medical-reasoning-verification](skills/scaling-medical-reasoning-verification/SKILL.md) | > | 191 |
| [st-raptor-agentic-system-semi-structured](skills/st-raptor-agentic-system-semi-structured/SKILL.md) | Agentic system for answering questions about semi-structured tables using tree-based structural modeling and multi-step ... | 222 |
| [standardizing-longitudinal-radiology-report](skills/standardizing-longitudinal-radiology-report/SKILL.md) | Build LLM-based pipelines that automatically detect and classify longitudinal (temporal) changes in radiology reports. U... | 230 |
| [steuerllm-local-specialized-german](skills/steuerllm-local-specialized-german/SKILL.md) | Build domain-specialized LLM pipelines for formal-rule domains (tax law, legal, regulatory) using retrieval-augmented sy... | 203 |
| [syncabel-synthetic-contextualized-augmentation](skills/syncabel-synthetic-contextualized-augmentation/SKILL.md) | Generate synthetic training data for biomedical entity linking using LLM-based contextualized augmentation. Use when: 'g... | 193 |
| [synthagent-multi-agent-framework-realistic](skills/synthagent-multi-agent-framework-realistic/SKILL.md) | Build multi-agent pipelines that generate realistic synthetic patient profiles by integrating epidemiological data, medi... | 298 |
| [the-clef-2026-finmmeval-lab](skills/the-clef-2026-finmmeval-lab/SKILL.md) | Build multilingual, multimodal financial AI evaluation pipelines using the FinMMEval framework. Covers financial exam QA... | 246 |
| [training-data-selection-gradient](skills/training-data-selection-gradient/SKILL.md) | Implement Orthogonal Gradient Selection (OGS) for efficient domain adaptation of LLMs—select training data whose gradien... | 196 |
| [unikie-bench-benchmarking-multimodal-key](skills/unikie-bench-benchmarking-multimodal-key/SKILL.md) | Extract structured key information from document images using schema-guided prompting for LMMs. Builds KIE pipelines tha... | 292 |
| [vihermes-graph-grounded-multihop-question](skills/vihermes-graph-grounded-multihop-question/SKILL.md) | Build graph-grounded multihop QA systems over regulatory and hierarchically structured documents. Combines vector simila... | 264 |
| [vlm-guided-iterative-refinement-surgical](skills/vlm-guided-iterative-refinement-surgical/SKILL.md) | Build iterative VLM-guided refinement pipelines for image segmentation tasks, especially surgical/medical imagery. Uses ... | 255 |
| [xai-clip-roi-guided-perturbation-framework](skills/xai-clip-roi-guided-perturbation-framework/SKILL.md) | Build ROI-guided perturbation pipelines for explainable medical image segmentation using CLIP embeddings. Generates boun... | 226 |

---

## Knowledge Graphs

**38 skills**

| Skill | Description | Lines |
|-------|-------------|-------|
| [a2rag-adaptive-agentic-graph](skills/a2rag-adaptive-agentic-graph/SKILL.md) | Build adaptive, cost-aware Graph-RAG pipelines that route queries through escalating retrieval stages (local -> bridge -... | 229 |
| [agent2agent-threats-safety-critical-assistants](skills/agent2agent-threats-safety-critical-assistants/SKILL.md) | Threat model multi-agent LLM systems using the AgentHeLLM framework -- formally separating asset identification from att... | 207 |
| [agentskiller-scaling-generalist-agent](skills/agentskiller-scaling-generalist-agent/SKILL.md) | Synthesize multi-turn agent interaction data across semantically linked domains using DAG-based pipelines, domain ontolo... | 178 |
| [autonomous-chain-of-thought-distillation-graph-bas](skills/autonomous-chain-of-thought-distillation-graph-bas/SKILL.md) | Implement FraudCoT-style graph-aware chain-of-thought distillation for fraud detection on text-attributed graphs. Combin... | 199 |
| [better-generalizing-unseen-concepts](skills/better-generalizing-unseen-concepts/SKILL.md) | Build biomedical concept recognition systems that generalize to unseen ontology concepts using hierarchical indexing and... | 143 |
| [beyond-blame-rethinking-szz](skills/beyond-blame-rethinking-szz/SKILL.md) | Identify bug-inducing commits using temporal knowledge graph search beyond git blame. Use when: 'find what commit introd... | 154 |
| [breaking-static-graph-context-aware](skills/breaking-static-graph-context-aware/SKILL.md) | Build query-adaptive knowledge graph retrieval systems using CatRAG's context-aware traversal. Transforms static KG-base... | 170 |
| [core-comprehensive-ontological-relation](skills/core-comprehensive-ontological-relation/SKILL.md) | Detect and prevent semantic collapse in LLM outputs — where models fabricate spurious relationships between unrelated co... | 216 |
| [embodied-task-planning-graph-informed](skills/embodied-task-planning-graph-informed/SKILL.md) | Structure long-horizon task planning using graph-based memory and bounded lookahead. Use when asked to: 'plan a multi-st... | 179 |
| [evaluation-entity-matching-recommender](skills/evaluation-entity-matching-recommender/SKILL.md) | Build and evaluate cross-dataset entity matching pipelines for recommender systems. Implements the Reddit-Amazon-EM meth... | 182 |
| [flyaoc-evaluating-agentic-ontology](skills/flyaoc-evaluating-agentic-ontology/SKILL.md) | Build multi-agent systems for end-to-end ontology curation from scientific literature. Applies FlyAOC's agent architectu... | 184 |
| [gamms-graph-based-adversarial](skills/gamms-graph-based-adversarial/SKILL.md) | > | 305 |
| [generative-ontology-structured-knowledge](skills/generative-ontology-structured-knowledge/SKILL.md) | > | 222 |
| [graph-anchored-knowledge-indexing-retrieval-augmen](skills/graph-anchored-knowledge-indexing-retrieval-augmen/SKILL.md) | Build iterative RAG pipelines that construct evolving knowledge graphs to anchor retrieval across multiple hops. Use whe... | 221 |
| [graph-based-agent-memory-taxonomy](skills/graph-based-agent-memory-taxonomy/SKILL.md) | Design and implement graph-based memory systems for LLM agents following the extraction-storage-retrieval-evolution life... | 279 |
| [graphagents-knowledge-graph-guided-agentic](skills/graphagents-knowledge-graph-guided-agentic/SKILL.md) | Build multi-agent pipelines that use knowledge graphs to guide LLM reasoning across domains. Agents specialize in proble... | 185 |
| [graphdancer-training-explore-reason](skills/graphdancer-training-explore-reason/SKILL.md) | Build agentic graph-exploration systems where an LLM navigates heterogeneous knowledge graphs through interleaved reason... | 243 |
| [graphseek-next-generation-graph-analytics](skills/graphseek-next-generation-graph-analytics/SKILL.md) | Build LLM-powered graph analytics systems using the GraphSeek two-plane architecture: a Semantic Catalog for planning ov... | 151 |
| [how-much-reasoning-retrieval-augmented](skills/how-much-reasoning-retrieval-augmented/SKILL.md) | Build contamination-aware hybrid RAG evaluation pipelines that couple knowledge graphs with text retrieval for multi-hop... | 178 |
| [hugrag-hierarchical-causal-knowledge](skills/hugrag-hierarchical-causal-knowledge/SKILL.md) | Build hierarchical causal knowledge graphs for RAG pipelines that suppress spurious correlations and enable cross-docume... | 168 |
| [kg-craft-knowledge-graph-based-contrastive](skills/kg-craft-knowledge-graph-based-contrastive/SKILL.md) | > | 197 |
| [knowledge-graphs-implicit-reward](skills/knowledge-graphs-implicit-reward/SKILL.md) | Build compositional reasoning systems that use knowledge graph paths as reward signals to ground LLM reasoning in verifi... | 168 |
| [koral-knowledge-graph-guided](skills/koral-knowledge-graph-guided/SKILL.md) | Build Knowledge Graph-guided LLM reasoning pipelines for operational telemetry analysis. Combines a Literature KG (extra... | 247 |
| [learning-decode-against-compositional](skills/learning-decode-against-compositional/SKILL.md) | Detect and mitigate compositional hallucinations in video multimodal LLM outputs using triple-pathway contrastive decodi... | 284 |
| [lec-kg-llm-embedding-collaborative-framework](skills/lec-kg-llm-embedding-collaborative-framework/SKILL.md) | Build domain-specific knowledge graphs from unstructured text using an iterative LLM + embedding validation loop. Combin... | 186 |
| [medspeak-knowledge-graph-aided-asr](skills/medspeak-knowledge-graph-aided-asr/SKILL.md) | Build knowledge-graph-aided ASR error correction pipelines for medical speech, using phonetic similarity + semantic retr... | 262 |
| [multi-targeted-graph-backdoor-attack](skills/multi-targeted-graph-backdoor-attack/SKILL.md) | Implement and analyze multi-targeted backdoor attacks on Graph Neural Networks (GNNs) using subgraph injection. Use when... | 193 |
| [nag-unified-native-architecture](skills/nag-unified-native-architecture/SKILL.md) | Encode graph structure directly into LM attention masks and positional IDs instead of using external GNN encoders. Use w... | 204 |
| [ontology-to-tools-compilation-executable-semantic-](skills/ontology-to-tools-compilation-executable-semantic/SKILL.md) | Compile domain ontologies (OWL/RDFS/JSON-LD schemas) into executable tool interfaces with embedded semantic constraints,... | 192 |
| [pathreasoner-r1-instilling-structured-reasoning](skills/pathreasoner-r1-instilling-structured-reasoning/SKILL.md) | Build knowledge-graph-guided structured reasoning pipelines for vision-language models in computational pathology. Imple... | 296 |
| [phenolip-integrating-phenotype-ontology](skills/phenolip-integrating-phenotype-ontology/SKILL.md) | Build phenotype-aware medical vision-language models by integrating ontology knowledge graphs into CLIP-style pretrainin... | 234 |
| [prograph-r1-progress-aware-reinforcement-learning](skills/prograph-r1-progress-aware-reinforcement-learning/SKILL.md) | Build progress-aware GraphRAG agents that traverse knowledge graphs with structure-aware hypergraph retrieval and dense ... | 167 |
| [rpo-rag-aligning-small-relation-aware](skills/rpo-rag-aligning-small-relation-aware/SKILL.md) | Build knowledge-graph-grounded RAG pipelines that align small LLMs (under 8B params) with relation-aware preference opti... | 259 |
| [temp-r1-unified-autonomous-agent](skills/temp-r1-unified-autonomous-agent/SKILL.md) | Build autonomous agents that answer complex temporal questions over knowledge graphs or time-stamped datasets using stru... | 200 |
| [toward-culturally-aligned-ontology-guided](skills/toward-culturally-aligned-ontology-guided/SKILL.md) | Ontology-guided multi-agent reasoning for culturally aligned LLM outputs. Use when building systems that must respect cu... | 190 |
| [use-graph-it-needs](skills/use-graph-it-needs/SKILL.md) | Implement adaptive RAG pipelines that route queries to dense retrieval, graph-based retrieval, or a weighted fusion base... | 254 |
| [vihermes-graph-grounded-multihop-question](skills/vihermes-graph-grounded-multihop-question/SKILL.md) | Build graph-grounded multihop QA systems over regulatory and hierarchically structured documents. Combines vector simila... | 264 |
| [vulread-knowledge-graph-guided-software-vulnerabil](skills/vulread-knowledge-graph-guided-software-vulnerabil/SKILL.md) | CWE-guided vulnerability reasoning and detection using knowledge-graph-structured analysis. Analyzes source code for sec... | 212 |

---

## All Skills (Alphabetical)

Complete list of all 1033 skills.

| Skill | Categories | Lines |
|-------|------------|-------|
| [1100-high-efficiency-visual-adapter-complex](skills/1100-high-efficiency-visual-adapter-complex/SKILL.md) | Fine-tuning & Training, Multimodal, Efficiency & Optimization | 251 |
| [3-secbench-large-scale-evaluation-suite-security](skills/3-secbench-large-scale-evaluation-suite-security/SKILL.md) | Security & Safety, Evaluation & Benchmarking, Agentic Systems +3 | 182 |
| [3d-space-as-scratchpad-editable](skills/3d-space-as-scratchpad-editable/SKILL.md) | Reasoning & Chain-of-Thought, Multimodal, Agentic Systems +1 | 172 |
| [a-mapreduce-executing-wide-search](skills/a-mapreduce-executing-wide-search/SKILL.md) | RAG & Retrieval, Agentic Systems, Explainability | 196 |
| [a-rag-scaling-agentic-retrieval-augmented](skills/a-rag-scaling-agentic-retrieval-augmented/SKILL.md) | RAG & Retrieval, Agentic Systems | 253 |
| [a2-llm-end-to-end-conversational-audio-avatar](skills/a2-llm-end-to-end-conversational-audio-avatar/SKILL.md) | Multimodal, Data Processing | 244 |
| [a2rag-adaptive-agentic-graph](skills/a2rag-adaptive-agentic-graph/SKILL.md) | RAG & Retrieval, Knowledge Graphs, Agentic Systems +3 | 229 |
| [aacr-bench-evaluating-automatic-code](skills/aacr-bench-evaluating-automatic-code/SKILL.md) | RAG & Retrieval, Code & Software Engineering, Evaluation & Benchmarking | 187 |
| [accelerating-social-science-research](skills/accelerating-social-science-research/SKILL.md) | RAG & Retrieval, Agentic Systems, Efficiency & Optimization +1 | 177 |
| [acegrpo-adaptive-curriculum-group](skills/acegrpo-adaptive-curriculum-group/SKILL.md) | Fine-tuning & Training, Agentic Systems, Efficiency & Optimization +1 | 220 |
| [adaptbpe-general-purpose-specialized](skills/adaptbpe-general-purpose-specialized/SKILL.md) | Other | 202 |
| [adaptive-confidence-gating-multi-agent](skills/adaptive-confidence-gating-multi-agent/SKILL.md) | Multi-Agent Systems, Code & Software Engineering, Agentic Systems | 184 |
| [adareasoner-dynamic-tool-orchestration](skills/adareasoner-dynamic-tool-orchestration/SKILL.md) | Multi-Agent Systems, Reasoning & Chain-of-Thought, Data Processing | 168 |
| [adatsq-pushing-pareto-frontier](skills/adatsq-pushing-pareto-frontier/SKILL.md) | Other | 231 |
| [addressing-explainability-generative-ai](skills/addressing-explainability-generative-ai/SKILL.md) | Code & Software Engineering, Prompt Engineering, Data Processing +1 | 222 |
| [adoption-use-at-academic](skills/adoption-use-at-academic/SKILL.md) | Other | 389 |
| [aegis-governance-integrity-security](skills/aegis-governance-integrity-security/SKILL.md) | Security & Safety, Evaluation & Benchmarking, Multimodal +2 | 237 |
| [aero-autonomous-evolutionary-reasoning](skills/aero-autonomous-evolutionary-reasoning/SKILL.md) | Reasoning & Chain-of-Thought, Agentic Systems | 160 |
| [affective-flow-emotional-support](skills/affective-flow-emotional-support/SKILL.md) | RAG & Retrieval, Agentic Systems, Efficiency & Optimization | 216 |
| [agent-based-software-artifact-evaluation](skills/agent-based-software-artifact-evaluation/SKILL.md) | RAG & Retrieval, Evaluation & Benchmarking, Agentic Systems | 203 |
| [agent-fence-mapping-security-vulnerabilities](skills/agent-fence-mapping-security-vulnerabilities/SKILL.md) | Security & Safety, Agentic Systems | 211 |
| [agent-primitives-reusable-latent](skills/agent-primitives-reusable-latent/SKILL.md) | Multi-Agent Systems, Agentic Systems, Data Processing | 252 |
| [agent2agent-threats-safety-critical-assistants](skills/agent2agent-threats-safety-critical-assistants/SKILL.md) | Multi-Agent Systems, Security & Safety, Knowledge Graphs +2 | 207 |
| [agentark-distilling-multi-agent-intelligence](skills/agentark-distilling-multi-agent-intelligence/SKILL.md) | Multi-Agent Systems, Reasoning & Chain-of-Thought, Fine-tuning & Training +4 | 199 |
| [agentcgroup-understanding-controlling-os](skills/agentcgroup-understanding-controlling-os/SKILL.md) | Memory & Context, Agentic Systems | 231 |
| [agentcpm-report-interleaving-drafting-deepening](skills/agentcpm-report-interleaving-drafting-deepening/SKILL.md) | Agentic Systems | 232 |
| [agentdog-diagnostic-guardrail-framework](skills/agentdog-diagnostic-guardrail-framework/SKILL.md) | Security & Safety, Agentic Systems | 340 |
| [agentdrive-open-benchmark-dataset](skills/agentdrive-open-benchmark-dataset/SKILL.md) | Security & Safety, Reasoning & Chain-of-Thought, Evaluation & Benchmarking +3 | 284 |
| [agentic-ai-healthcare-medicine](skills/agentic-ai-healthcare-medicine/SKILL.md) | Multi-Agent Systems, Evaluation & Benchmarking, Agentic Systems +1 | 274 |
| [agentic-reinforcement-learning-empowers](skills/agentic-reinforcement-learning-empowers/SKILL.md) | Multi-Agent Systems, RAG & Retrieval, Reasoning & Chain-of-Thought +3 | 242 |
| [agentic-very-long-video](skills/agentic-very-long-video/SKILL.md) | RAG & Retrieval, Reasoning & Chain-of-Thought, Multimodal +1 | 240 |
| [agenticpay-multi-agent-negotiation-system](skills/agenticpay-multi-agent-negotiation-system/SKILL.md) | Multi-Agent Systems, Agentic Systems, Data Processing | 226 |
| [agenticscr-an-autonomous-agentic](skills/agenticscr-an-autonomous-agentic/SKILL.md) | Agentic Systems | 177 |
| [agenticsimlaw-juvenile-courtroom-multi-agent](skills/agenticsimlaw-juvenile-courtroom-multi-agent/SKILL.md) | Multi-Agent Systems, Security & Safety, Reasoning & Chain-of-Thought +3 | 181 |
| [agentskiller-scaling-generalist-agent](skills/agentskiller-scaling-generalist-agent/SKILL.md) | Fine-tuning & Training, Knowledge Graphs, Agentic Systems +1 | 178 |
| [agentsm-semantic-memory-agentic](skills/agentsm-semantic-memory-agentic/SKILL.md) | Memory & Context, Agentic Systems, Data Processing | 213 |
| [agentstepper-interactive-debugging-software](skills/agentstepper-interactive-debugging-software/SKILL.md) | Code & Software Engineering, Agentic Systems, Prompt Engineering | 197 |
| [agentsys-secure-dynamic-agents](skills/agentsys-secure-dynamic-agents/SKILL.md) | Agentic Systems | 191 |
| [agenttrace-structured-logging-framework](skills/agenttrace-structured-logging-framework/SKILL.md) | RAG & Retrieval, Security & Safety, Reasoning & Chain-of-Thought +1 | 152 |
| [agentxray-white-boxing-agentic-systems](skills/agentxray-white-boxing-agentic-systems/SKILL.md) | RAG & Retrieval, Agentic Systems, Data Processing +1 | 168 |
| [agyn-multi-agent-system-team-based](skills/agyn-multi-agent-system-team-based/SKILL.md) | Multi-Agent Systems, RAG & Retrieval, Code & Software Engineering +1 | 202 |
| [ai-agent-for-reverseengineering](skills/ai-agent-for-reverseengineering/SKILL.md) | Agentic Systems | 205 |
| [ai-agent-systems-supply](skills/ai-agent-systems-supply/SKILL.md) | Agentic Systems | 277 |
| [ai-my-values-user](skills/ai-my-values-user/SKILL.md) | Agentic Systems, Data Processing, Explainability | 236 |
| [aiano-enhancing-information-retrieval](skills/aiano-enhancing-information-retrieval/SKILL.md) | RAG & Retrieval, Fine-tuning & Training, Efficiency & Optimization +1 | 161 |
| [aicd-bench-challenging-benchmark](skills/aicd-bench-challenging-benchmark/SKILL.md) | Security & Safety, Evaluation & Benchmarking, Data Processing +1 | 211 |
| [aidev-studying-ai-coding](skills/aidev-studying-ai-coding/SKILL.md) | Multi-Agent Systems, Evaluation & Benchmarking, Agentic Systems | 195 |
| [alertguardian-intelligent-alert-life-cycle](skills/alertguardian-intelligent-alert-life-cycle/SKILL.md) | Other | 201 |
| [alienlm-alienization-api-boundary-privacy](skills/alienlm-alienization-api-boundary-privacy/SKILL.md) | Prompt Engineering, Data Processing | 202 |
| [alignagent-adaptive-learner-intelligence](skills/alignagent-adaptive-learner-intelligence/SKILL.md) | Agentic Systems | 319 |
| [aligncoder-aligning-retrieval-target](skills/aligncoder-aligning-retrieval-target/SKILL.md) | RAG & Retrieval | 206 |
| [alignment-drift-multimodal-two-phase](skills/alignment-drift-multimodal-two-phase/SKILL.md) | Multimodal | 312 |
| [aligntune-modular-toolkit-post-training](skills/aligntune-modular-toolkit-post-training/SKILL.md) | Fine-tuning & Training | 287 |
| [alphaface-high-fidelity-real-time](skills/alphaface-high-fidelity-real-time/SKILL.md) | RAG & Retrieval | 225 |
| [alrm-agentic-robotic-manipulation](skills/alrm-agentic-robotic-manipulation/SKILL.md) | Agentic Systems | 131 |
| [ama-adaptive-memory-multi-agent](skills/ama-adaptive-memory-multi-agent/SKILL.md) | Multi-Agent Systems, RAG & Retrieval, Memory & Context +2 | 224 |
| [amem4rec-leveraging-cross-user-similarity](skills/amem4rec-leveraging-cross-user-similarity/SKILL.md) | RAG & Retrieval, Fine-tuning & Training, Memory & Context +1 | 273 |
| [an-cost-efficient-agentic-framework](skills/an-cost-efficient-agentic-framework/SKILL.md) | Security & Safety, Code & Software Engineering, Reasoning & Chain-of-Thought +3 | 202 |
| [analyticsgpt-workflow-scientometric-question](skills/analyticsgpt-workflow-scientometric-question/SKILL.md) | RAG & Retrieval, Evaluation & Benchmarking, Agentic Systems +2 | 306 |
| [annotation-free-spacecraft-detection](skills/annotation-free-spacecraft-detection/SKILL.md) | Data Processing | 229 |
| [anonymization-enhanced-privacy-protection-mobile-g](skills/anonymization-enhanced-privacy-protection-mobile-g/SKILL.md) | Agentic Systems, Data Processing | 178 |
| [aorchestra-automating-sub-agent-creation](skills/aorchestra-automating-sub-agent-creation/SKILL.md) | Multi-Agent Systems, Agentic Systems, Data Processing | 203 |
| [apex-agents](skills/apex-agents/SKILL.md) | Agentic Systems | 223 |
| [aqascore-evaluating-semantic-alignment](skills/aqascore-evaluating-semantic-alignment/SKILL.md) | Evaluation & Benchmarking | 210 |
| [ar-map-autoregressive-implicit-teachers](skills/ar-map-autoregressive-implicit-teachers/SKILL.md) | Other | 164 |
| [ar-omni-unified-autoregressive-any-to-any](skills/ar-omni-unified-autoregressive-any-to-any/SKILL.md) | Other | 225 |
| [are-open-weight-ready-social](skills/are-open-weight-ready-social/SKILL.md) | Evaluation & Benchmarking, Prompt Engineering, Data Processing | 264 |
| [arkeval-benchmarking-evaluating-automated](skills/arkeval-benchmarking-evaluating-automated/SKILL.md) | RAG & Retrieval, Code & Software Engineering, Evaluation & Benchmarking | 196 |
| [artificial-intelligence-open-source](skills/artificial-intelligence-open-source/SKILL.md) | Other | 210 |
| [asa-training-free-representation-engineering](skills/asa-training-free-representation-engineering/SKILL.md) | Fine-tuning & Training, Agentic Systems, Data Processing | 170 |
| [assessing-business-process-modeling](skills/assessing-business-process-modeling/SKILL.md) | RAG & Retrieval, Evaluation & Benchmarking | 229 |
| [assessing-domain-level-susceptibility-emergent](skills/assessing-domain-level-susceptibility-emergent/SKILL.md) | Evaluation & Benchmarking | 287 |
| [assessing-quality-mental-health](skills/assessing-quality-mental-health/SKILL.md) | Evaluation & Benchmarking | 374 |
| [assessment-generative-named-entity](skills/assessment-generative-named-entity/SKILL.md) | Evaluation & Benchmarking, Fine-tuning & Training, Prompt Engineering +2 | 245 |
| [astra-automated-synthesis-of](skills/astra-automated-synthesis-of/SKILL.md) | Other | 220 |
| [atomic-information-flow-network](skills/atomic-information-flow-network/SKILL.md) | Other | 197 |
| [attention-based-offline-reinforcement-learning](skills/attention-based-offline-reinforcement-learning/SKILL.md) | Memory & Context | 203 |
| [attn-gs-attention-guided-context-compression](skills/attn-gs-attention-guided-context-compression/SKILL.md) | Memory & Context, Prompt Engineering, Efficiency & Optimization +2 | 158 |
| [audiorouter-data-audio-understanding](skills/audiorouter-data-audio-understanding/SKILL.md) | Reasoning & Chain-of-Thought, Multimodal, Agentic Systems +2 | 249 |
| [audit-after-segmentation-reference-free](skills/audit-after-segmentation-reference-free/SKILL.md) | Evaluation & Benchmarking, Multimodal, Agentic Systems +1 | 251 |
| [automated-customization-enterprise-code](skills/automated-customization-enterprise-code/SKILL.md) | RAG & Retrieval, Code & Software Engineering, Fine-tuning & Training +1 | 153 |
| [automated-multiple-mini-interview](skills/automated-multiple-mini-interview/SKILL.md) | Multi-Agent Systems, Evaluation & Benchmarking, Agentic Systems +2 | 189 |
| [automated-rubrics-reliable-evaluation](skills/automated-rubrics-reliable-evaluation/SKILL.md) | Multi-Agent Systems, RAG & Retrieval, Evaluation & Benchmarking +3 | 180 |
| [automated-structural-testing-llm-based](skills/automated-structural-testing-llm-based/SKILL.md) | Agentic Systems | 242 |
| [automating-computational-reproducibility-social](skills/automating-computational-reproducibility-social/SKILL.md) | RAG & Retrieval, Code & Software Engineering, Reasoning & Chain-of-Thought +2 | 189 |
| [autonomous-business-system-neuro-symbolic](skills/autonomous-business-system-neuro-symbolic/SKILL.md) | Agentic Systems | 249 |
| [autonomous-chain-of-thought-distillation-graph-bas](skills/autonomous-chain-of-thought-distillation-graph-bas/SKILL.md) | Reasoning & Chain-of-Thought, Evaluation & Benchmarking, Fine-tuning & Training +5 | 199 |
| [autonomous-data-processing-meta-agents](skills/autonomous-data-processing-meta-agents/SKILL.md) | Multi-Agent Systems, Agentic Systems, Data Processing | 216 |
| [autonomous-multi-agent-ai-high-throughput](skills/autonomous-multi-agent-ai-high-throughput/SKILL.md) | Multi-Agent Systems, Agentic Systems | 189 |
| [autoregressive-yet-revisable-decoding-revision](skills/autoregressive-yet-revisable-decoding-revision/SKILL.md) | Multimodal | 201 |
| [avenir-web-human-experience-imitating-multimodal-w](skills/avenir-web-human-experience-imitating-multimodal-w/SKILL.md) | Memory & Context, Multimodal, Agentic Systems | 375 |
| [avere-improving-audiovisual-emotion](skills/avere-improving-audiovisual-emotion/SKILL.md) | Reasoning & Chain-of-Thought, Evaluation & Benchmarking, Multimodal +3 | 177 |
| [bagging-based-merging-robust-general](skills/bagging-based-merging-robust-general/SKILL.md) | Other | 184 |
| [bass-benchmarking-audio-lms](skills/bass-benchmarking-audio-lms/SKILL.md) | Multi-Agent Systems, Reasoning & Chain-of-Thought, Evaluation & Benchmarking +2 | 260 |
| [batcoder-self-supervised-bidirectional-code-docume](skills/batcoder-self-supervised-bidirectional-code-docume/SKILL.md) | Code & Software Engineering, NLP & Text | 207 |
| [bayesflow-probability-inference-framework](skills/bayesflow-probability-inference-framework/SKILL.md) | Agentic Systems, Prompt Engineering, Efficiency & Optimization +1 | 204 |
| [bear-beam-search-aware-optimization-recommendation](skills/bear-beam-search-aware-optimization-recommendation/SKILL.md) | RAG & Retrieval, Fine-tuning & Training, Efficiency & Optimization | 220 |
| [behavioural-representational-evaluation-goal-direc](skills/behavioural-representational-evaluation-goal-direc/SKILL.md) | Evaluation & Benchmarking, Agentic Systems, Explainability | 172 |
| [benchmarking-abap-code-generation](skills/benchmarking-abap-code-generation/SKILL.md) | Code & Software Engineering, Evaluation & Benchmarking | 183 |
| [benchmarking-pairwise-causal-discovery](skills/benchmarking-pairwise-causal-discovery/SKILL.md) | Evaluation & Benchmarking, Explainability | 204 |
| [benchmarking-reward-hack-detection](skills/benchmarking-reward-hack-detection/SKILL.md) | Evaluation & Benchmarking, Agentic Systems | 190 |
| [benchmarking-text-to-python-against-text-to-sql](skills/benchmarking-text-to-python-against-text-to-sql/SKILL.md) | Reasoning & Chain-of-Thought, Evaluation & Benchmarking, NLP & Text | 188 |
| [benchmarking-uncertainty-calibration-long-form](skills/benchmarking-uncertainty-calibration-long-form/SKILL.md) | Evaluation & Benchmarking | 210 |
| [benchmarking-zero-shot-few-shot-phishing](skills/benchmarking-zero-shot-few-shot-phishing/SKILL.md) | Security & Safety, Evaluation & Benchmarking, Prompt Engineering | 215 |
| [better-as-generators-than](skills/better-as-generators-than/SKILL.md) | Fine-tuning & Training, Efficiency & Optimization, Data Processing +1 | 178 |
| [better-generalizing-unseen-concepts](skills/better-generalizing-unseen-concepts/SKILL.md) | Evaluation & Benchmarking, Fine-tuning & Training, Knowledge Graphs +3 | 143 |
| [beyond-accuracy-cognitive-load](skills/beyond-accuracy-cognitive-load/SKILL.md) | Multi-Agent Systems, Agentic Systems, Efficiency & Optimization +1 | 218 |
| [beyond-alignment-expanding-reasoning](skills/beyond-alignment-expanding-reasoning/SKILL.md) | Reasoning & Chain-of-Thought, Evaluation & Benchmarking, Fine-tuning & Training +2 | 250 |
| [beyond-blame-rethinking-szz](skills/beyond-blame-rethinking-szz/SKILL.md) | RAG & Retrieval, Code & Software Engineering, Knowledge Graphs | 154 |
| [beyond-confidence-rhythms-reasoning](skills/beyond-confidence-rhythms-reasoning/SKILL.md) | RAG & Retrieval, Reasoning & Chain-of-Thought, Evaluation & Benchmarking +1 | 169 |
| [beyond-function-level-analysis-context-aware](skills/beyond-function-level-analysis-context-aware/SKILL.md) | Other | 170 |
| [beyond-holistic-scores-automatic](skills/beyond-holistic-scores-automatic/SKILL.md) | Evaluation & Benchmarking, Prompt Engineering | 201 |
| [beyond-imitation-reinforcement-learning](skills/beyond-imitation-reinforcement-learning/SKILL.md) | Other | 257 |
| [beyond-instrumental-substitutive-paradigms](skills/beyond-instrumental-substitutive-paradigms/SKILL.md) | Fine-tuning & Training, Prompt Engineering | 189 |
| [beyond-needles-illusion-decoupled](skills/beyond-needles-illusion-decoupled/SKILL.md) | RAG & Retrieval, Evaluation & Benchmarking, Memory & Context +1 | 191 |
| [beyond-prompting-robust-contextual](skills/beyond-prompting-robust-contextual/SKILL.md) | Prompt Engineering | 220 |
| [beyond-single-perspective-text](skills/beyond-single-perspective-text/SKILL.md) | Evaluation & Benchmarking, NLP & Text | 195 |
| [beyond-speedup-utilizing](skills/beyond-speedup-utilizing/SKILL.md) | Reasoning & Chain-of-Thought, Evaluation & Benchmarking, Memory & Context +1 | 247 |
| [beyond-static-cropping-layer-adaptive](skills/beyond-static-cropping-layer-adaptive/SKILL.md) | Other | 209 |
| [beyond-superficial-unlearning-sharpness-aware](skills/beyond-superficial-unlearning-sharpness-aware/SKILL.md) | Evaluation & Benchmarking, Fine-tuning & Training, Multimodal | 269 |
| [beyond-translation-cross-cultural-meme](skills/beyond-translation-cross-cultural-meme/SKILL.md) | Multimodal, Data Processing, NLP & Text | 170 |
| [bi-directional-bias-attribution-debiasing](skills/bi-directional-bias-attribution-debiasing/SKILL.md) | Fine-tuning & Training, Prompt Engineering, Explainability | 184 |
| [bias-ear-listener-assessing](skills/bias-ear-listener-assessing/SKILL.md) | Evaluation & Benchmarking, Multimodal | 182 |
| [biases-blind-spot-detecting](skills/biases-blind-spot-detecting/SKILL.md) | Reasoning & Chain-of-Thought, Data Processing | 194 |
| [biasscope-automated-detection-bias](skills/biasscope-automated-detection-bias/SKILL.md) | Security & Safety, Evaluation & Benchmarking, Data Processing | 206 |
| [binaryppo-policy-optimization-binary](skills/binaryppo-policy-optimization-binary/SKILL.md) | Security & Safety, Fine-tuning & Training, Efficiency & Optimization +1 | 173 |
| [bioace-automated-framework-biomedical](skills/bioace-automated-framework-biomedical/SKILL.md) | Domain-Specific | 238 |
| [birdturk-adaptation-bird-text-to-sql](skills/birdturk-adaptation-bird-text-to-sql/SKILL.md) | Reasoning & Chain-of-Thought, Evaluation & Benchmarking, Agentic Systems +2 | 242 |
| [blind-gods-broken-screens](skills/blind-gods-broken-screens/SKILL.md) | Multi-Agent Systems, Security & Safety, Memory & Context +3 | 219 |
| [borp-bootstrapped-regression-probing](skills/borp-bootstrapped-regression-probing/SKILL.md) | Other | 190 |
| [breaking-protocol-security-analysis](skills/breaking-protocol-security-analysis/SKILL.md) | Security & Safety, Prompt Engineering | 264 |
| [breaking-static-graph-context-aware](skills/breaking-static-graph-context-aware/SKILL.md) | RAG & Retrieval, Knowledge Graphs, Data Processing | 170 |
| [bridging-academia-industry-comprehensive](skills/bridging-academia-industry-comprehensive/SKILL.md) | Evaluation & Benchmarking, Efficiency & Optimization, Data Processing +1 | 264 |
| [bridging-arithmetic-gap-cognitive](skills/bridging-arithmetic-gap-cognitive/SKILL.md) | Reasoning & Chain-of-Thought, Evaluation & Benchmarking, Data Processing +1 | 222 |
| [bridging-lexical-ambiguity-vision](skills/bridging-lexical-ambiguity-vision/SKILL.md) | Reasoning & Chain-of-Thought, Multimodal, Data Processing | 188 |
| [bridging-modality-gap-roadside](skills/bridging-modality-gap-roadside/SKILL.md) | Reasoning & Chain-of-Thought, Fine-tuning & Training, Multimodal +3 | 211 |
| [bridging-online-offline-rl](skills/bridging-online-offline-rl/SKILL.md) | Code & Software Engineering | 160 |
| [bridging-semantic-chasm-synergistic](skills/bridging-semantic-chasm-synergistic/SKILL.md) | Other | 178 |
| [c-mop-integrating-momentum-boundary-aware](skills/c-mop-integrating-momentum-boundary-aware/SKILL.md) | Prompt Engineering, Efficiency & Optimization | 179 |
| [c2rope-causal-continuous-rotary-positional](skills/c2rope-causal-continuous-rotary-positional/SKILL.md) | Explainability | 235 |
| [c3box-clip-based-class-incremental-learning](skills/c3box-clip-based-class-incremental-learning/SKILL.md) | Evaluation & Benchmarking, Prompt Engineering, Data Processing | 215 |
| [calliope-tts-based-narrated-e-book](skills/calliope-tts-based-narrated-e-book/SKILL.md) | Multimodal | 178 |
| [cam-causality-based-analysis-framework](skills/cam-causality-based-analysis-framework/SKILL.md) | Multi-Agent Systems, Code & Software Engineering, Agentic Systems +3 | 153 |
| [can-clean-up-mess](skills/can-clean-up-mess/SKILL.md) | Data Processing | 164 |
| [can-implement-agent-based-odd-based](skills/can-implement-agent-based-odd-based/SKILL.md) | Reasoning & Chain-of-Thought, Agentic Systems, Data Processing +1 | 208 |
| [can-post-training-transform-causal](skills/can-post-training-transform-causal/SKILL.md) | Reasoning & Chain-of-Thought, Fine-tuning & Training, Data Processing +1 | 179 |
| [can-reasoning-be-trusted](skills/can-reasoning-be-trusted/SKILL.md) | Reasoning & Chain-of-Thought, Evaluation & Benchmarking, Data Processing | 203 |
| [can-small-handle-context-summarized](skills/can-small-handle-context-summarized/SKILL.md) | Evaluation & Benchmarking, Memory & Context, Prompt Engineering +2 | 254 |
| [can-truly-embody-human](skills/can-truly-embody-human/SKILL.md) | Evaluation & Benchmarking, Agentic Systems, Prompt Engineering | 206 |
| [can-vision-language-handle-long-context](skills/can-vision-language-handle-long-context/SKILL.md) | Code & Software Engineering, Memory & Context, Multimodal +2 | 185 |
| [can-we-classify-flaky](skills/can-we-classify-flaky/SKILL.md) | Reasoning & Chain-of-Thought, Evaluation & Benchmarking | 182 |
| [canonical-intermediate-representation-llm-based](skills/canonical-intermediate-representation-llm-based/SKILL.md) | Other | 237 |
| [capture-flags-family-based-evaluation](skills/capture-flags-family-based-evaluation/SKILL.md) | Evaluation & Benchmarking | 165 |
| [causal-perspective-enhancing-jailbreak-attack](skills/causal-perspective-enhancing-jailbreak-attack/SKILL.md) | Security & Safety, Prompt Engineering, Data Processing +1 | 175 |
| [causalt5k-diagnosing-informing-refusal](skills/causalt5k-diagnosing-informing-refusal/SKILL.md) | Reasoning & Chain-of-Thought, Explainability | 161 |
| [causaltad-injecting-causal-knowledge](skills/causaltad-injecting-causal-knowledge/SKILL.md) | Fine-tuning & Training, Data Processing, Explainability | 267 |
| [cgpt-cluster-guided-partial-tables](skills/cgpt-cluster-guided-partial-tables/SKILL.md) | RAG & Retrieval, Fine-tuning & Training | 230 |
| [chain-mindset-reasoning-adaptive](skills/chain-mindset-reasoning-adaptive/SKILL.md) | Reasoning & Chain-of-Thought | 177 |
| [chain-simulation-dual-mode-reasoning](skills/chain-simulation-dual-mode-reasoning/SKILL.md) | Reasoning & Chain-of-Thought, Data Processing | 175 |
| [chart-specification-structural-representations](skills/chart-specification-structural-representations/SKILL.md) | Other | 344 |
| [chatting-images-introspective-visual](skills/chatting-images-introspective-visual/SKILL.md) | Reasoning & Chain-of-Thought, Multimodal, Explainability | 186 |
| [chehab-rl-learning-optimize](skills/chehab-rl-learning-optimize/SKILL.md) | Code & Software Engineering, Efficiency & Optimization | 206 |
| [chipbench-next-step-benchmark-evaluating](skills/chipbench-next-step-benchmark-evaluating/SKILL.md) | Code & Software Engineering, Evaluation & Benchmarking, NLP & Text | 225 |
| [chunking-retrieval-re-ranking-empirical-evaluation](skills/chunking-retrieval-re-ranking-empirical-evaluation/SKILL.md) | RAG & Retrieval, Evaluation & Benchmarking, Efficiency & Optimization +1 | 230 |
| [chunkwise-lora-adaptive-sequence](skills/chunkwise-lora-adaptive-sequence/SKILL.md) | Other | 240 |
| [ci4a-semantic-component-interfaces](skills/ci4a-semantic-component-interfaces/SKILL.md) | Agentic Systems | 272 |
| [cimrag-cim-aware-domain-adaptive-noise-resilient](skills/cimrag-cim-aware-domain-adaptive-noise-resilient/SKILL.md) | RAG & Retrieval, Fine-tuning & Training, Efficiency & Optimization +1 | 227 |
| [closing-reasoning-gaps-clinical](skills/closing-reasoning-gaps-clinical/SKILL.md) | RAG & Retrieval, Reasoning & Chain-of-Thought, Agentic Systems +2 | 196 |
| [clustering-driven-memory-compression-on-device](skills/clustering-driven-memory-compression-on-device/SKILL.md) | Memory & Context, Efficiency & Optimization | 254 |
| [co-redteam-orchestrated-security-discovery](skills/co-redteam-orchestrated-security-discovery/SKILL.md) | Multi-Agent Systems, Security & Safety, Reasoning & Chain-of-Thought +2 | 197 |
| [code2world-gui-world-renderable](skills/code2world-gui-world-renderable/SKILL.md) | Multimodal | 292 |
| [codecircuit-inferring-llm-generated-code](skills/codecircuit-inferring-llm-generated-code/SKILL.md) | Code & Software Engineering, Reasoning & Chain-of-Thought, Evaluation & Benchmarking +1 | 195 |
| [codeocr-effectiveness-vision-code](skills/codeocr-effectiveness-vision-code/SKILL.md) | Multimodal, Efficiency & Optimization | 241 |
| [cofrgenet-continued-fraction-architectures](skills/cofrgenet-continued-fraction-architectures/SKILL.md) | Memory & Context, Efficiency & Optimization | 267 |
| [cognitive-platform-engineering-autonomous](skills/cognitive-platform-engineering-autonomous/SKILL.md) | Multi-Agent Systems, Reasoning & Chain-of-Thought, Agentic Systems +1 | 247 |
| [cognitively-diverse-multiple-choice-question](skills/cognitively-diverse-multiple-choice-question/SKILL.md) | Other | 177 |
| [colt-lightweight-multi-llm-collaboration](skills/colt-lightweight-multi-llm-collaboration/SKILL.md) | Multi-Agent Systems, Efficiency & Optimization | 190 |
| [comet-collaborative-memory-transformer](skills/comet-collaborative-memory-transformer/SKILL.md) | Memory & Context, Efficiency & Optimization | 199 |
| [commcp-multi-agent-coordination-llm-based](skills/commcp-multi-agent-coordination-llm-based/SKILL.md) | Multi-Agent Systems, Agentic Systems, Data Processing | 225 |
| [compact-hypercube-embeddings-fast](skills/compact-hypercube-embeddings-fast/SKILL.md) | RAG & Retrieval, Memory & Context | 192 |
| [compactrag-reducing-calls-token](skills/compactrag-reducing-calls-token/SKILL.md) | RAG & Retrieval | 194 |
| [compar-ia-french-governments](skills/compar-ia-french-governments/SKILL.md) | Evaluation & Benchmarking, Fine-tuning & Training, Data Processing | 310 |
| [comparing-ai-coding-agents](skills/comparing-ai-coding-agents/SKILL.md) | Agentic Systems | 207 |
| [compass-contrastive-learning-automated](skills/compass-contrastive-learning-automated/SKILL.md) | Code & Software Engineering, Evaluation & Benchmarking | 180 |
| [completing-missing-annotation-multi-agent](skills/completing-missing-annotation-multi-agent/SKILL.md) | Multi-Agent Systems, Evaluation & Benchmarking, Agentic Systems +1 | 251 |
| [comprehensive-comparison-rag-methods](skills/comprehensive-comparison-rag-methods/SKILL.md) | RAG & Retrieval, Efficiency & Optimization, Data Processing | 167 |
| [comprehensive-evaluation-software-engineering](skills/comprehensive-evaluation-software-engineering/SKILL.md) | Evaluation & Benchmarking | 285 |
| [computational-approach-visual-metonymy](skills/computational-approach-visual-metonymy/SKILL.md) | Reasoning & Chain-of-Thought, Evaluation & Benchmarking, Multimodal +2 | 179 |
| [conceptual-cultural-index-metric](skills/conceptual-cultural-index-metric/SKILL.md) | Evaluation & Benchmarking | 246 |
| [confounding-robust-continuous-control](skills/confounding-robust-continuous-control/SKILL.md) | Data Processing, Explainability | 205 |
| [consistency-meets-verification-enhancing](skills/consistency-meets-verification-enhancing/SKILL.md) | Evaluation & Benchmarking, Data Processing | 178 |
| [constitutional-spec-driven-development-enforcing](skills/constitutional-spec-driven-development-enforcing/SKILL.md) | Security & Safety, Code & Software Engineering | 258 |
| [constrained-process-maps-multi-agent](skills/constrained-process-maps-multi-agent/SKILL.md) | Multi-Agent Systems, Agentic Systems, Data Processing | 244 |
| [constructing-multi-label-hierarchical-classificati](skills/constructing-multi-label-hierarchical-classificati/SKILL.md) | Security & Safety, Data Processing | 226 |
| [context-augmented-code-generation-programming-know](skills/context-augmented-code-generation-programming-know/SKILL.md) | Other | 191 |
| [context-sensitive-pointer-analysis-arkts](skills/context-sensitive-pointer-analysis-arkts/SKILL.md) | Security & Safety | 223 |
| [contextevolve-multi-agent-context-compression](skills/contextevolve-multi-agent-context-compression/SKILL.md) | Multi-Agent Systems, Memory & Context, Agentic Systems +2 | 175 |
| [contextual-drag-errors-context](skills/contextual-drag-errors-context/SKILL.md) | RAG & Retrieval | 249 |
| [controlling-output-rankings-generative](skills/controlling-output-rankings-generative/SKILL.md) | RAG & Retrieval, Reasoning & Chain-of-Thought, Efficiency & Optimization | 245 |
| [conversation-non-verifiable-learning-self-evolving](skills/conversation-non-verifiable-learning-self-evolving/SKILL.md) | Agentic Systems | 241 |
| [convexbench-recognize-convex-functions](skills/convexbench-recognize-convex-functions/SKILL.md) | Reasoning & Chain-of-Thought | 161 |
| [cope-clipped-rope-as](skills/cope-clipped-rope-as/SKILL.md) | Fine-tuning & Training, Memory & Context | 215 |
| [cord-bridging-audio-text-reasoning](skills/cord-bridging-audio-text-reasoning/SKILL.md) | Reasoning & Chain-of-Thought, Fine-tuning & Training, Multimodal +2 | 238 |
| [core-comprehensive-ontological-relation](skills/core-comprehensive-ontological-relation/SKILL.md) | Reasoning & Chain-of-Thought, Evaluation & Benchmarking, Knowledge Graphs +1 | 216 |
| [core-ubiquitous-6g-intelligence](skills/core-ubiquitous-6g-intelligence/SKILL.md) | Multi-Agent Systems, Agentic Systems, Data Processing | 195 |
| [corefine-confidence-guided-self-refinement-adaptiv](skills/corefine-confidence-guided-self-refinement-adaptiv/SKILL.md) | Code & Software Engineering, Reasoning & Chain-of-Thought, Evaluation & Benchmarking | 176 |
| [corpusqa-10-million-token](skills/corpusqa-10-million-token/SKILL.md) | Evaluation & Benchmarking, Memory & Context, Agentic Systems +2 | 179 |
| [cost-aware-selection-text-classification](skills/cost-aware-selection-text-classification/SKILL.md) | Other | 174 |
| [cost-efficient-rag-entity-matching](skills/cost-efficient-rag-entity-matching/SKILL.md) | RAG & Retrieval, Efficiency & Optimization, Data Processing | 187 |
| [covagent-overcoming-30-curse](skills/covagent-overcoming-30-curse/SKILL.md) | RAG & Retrieval, Agentic Systems | 174 |
| [cowork-x-experience-optimized-co-evolution-multi-a](skills/cowork-x-experience-optimized-co-evolution-multi-a/SKILL.md) | Multi-Agent Systems, Agentic Systems, Efficiency & Optimization | 150 |
| [craft-calibrated-reasoning-answer-faithful](skills/craft-calibrated-reasoning-answer-faithful/SKILL.md) | RAG & Retrieval, Reasoning & Chain-of-Thought, Data Processing +1 | 192 |
| [creditaudit-2textnd-dimension-evaluation](skills/creditaudit-2textnd-dimension-evaluation/SKILL.md) | Evaluation & Benchmarking, Agentic Systems, Prompt Engineering +1 | 211 |
| [cross-lingual-stability-judges-under](skills/cross-lingual-stability-judges-under/SKILL.md) | Evaluation & Benchmarking, Data Processing | 199 |
| [ctelm-decoding-manipulating-embeddings](skills/ctelm-decoding-manipulating-embeddings/SKILL.md) | Explainability | 203 |
| [ctrlcot-dual-granularity-chain-of-thought-compress](skills/ctrlcot-dual-granularity-chain-of-thought-compress/SKILL.md) | Reasoning & Chain-of-Thought, Efficiency & Optimization | 157 |
| [cua-skill-develop-skills-computer](skills/cua-skill-develop-skills-computer/SKILL.md) | RAG & Retrieval, Agentic Systems | 297 |
| [culturally-grounded-personas-characterization](skills/culturally-grounded-personas-characterization/SKILL.md) | Other | 202 |
| [curate-train-refine-closed-loop-agentic-framework-](skills/curate-train-refine-closed-loop-agentic-framework/SKILL.md) | Fine-tuning & Training, Agentic Systems, Prompt Engineering +2 | 148 |
| [cure-curriculum-guided-multi-task-training](skills/cure-curriculum-guided-multi-task-training/SKILL.md) | Fine-tuning & Training, Multimodal, Data Processing +1 | 234 |
| [curiosity-driven-knowledge-retrieval](skills/curiosity-driven-knowledge-retrieval/SKILL.md) | RAG & Retrieval | 219 |
| [curp-codebook-based-continuous-user](skills/curp-codebook-based-continuous-user/SKILL.md) | Other | 170 |
| [cutting-gordian-knot-detecting](skills/cutting-gordian-knot-detecting/SKILL.md) | Security & Safety, Reasoning & Chain-of-Thought | 212 |
| [cve-factory-scaling-expert-level-agentic](skills/cve-factory-scaling-expert-level-agentic/SKILL.md) | Agentic Systems | 240 |
| [cvedrl-code-verifier-difficulty-aware](skills/cvedrl-code-verifier-difficulty-aware/SKILL.md) | RAG & Retrieval | 195 |
| [d-orca-dialogue-centric-optimization-robust](skills/d-orca-dialogue-centric-optimization-robust/SKILL.md) | Fine-tuning & Training, Multimodal, Efficiency & Optimization +2 | 228 |
| [d2quant-accurate-low-bit-post-training-weight](skills/d2quant-accurate-low-bit-post-training-weight/SKILL.md) | Fine-tuning & Training, Memory & Context, Efficiency & Optimization +1 | 229 |
| [dancing-chains-strategic-persuasion](skills/dancing-chains-strategic-persuasion/SKILL.md) | Other | 266 |
| [darl-encouraging-diverse-answers](skills/darl-encouraging-diverse-answers/SKILL.md) | RAG & Retrieval, Prompt Engineering | 286 |
| [dart-diffusion-inspired-speculative-decoding](skills/dart-diffusion-inspired-speculative-decoding/SKILL.md) | Efficiency & Optimization | 241 |
| [dart-ing-drift-dynamic-tracing](skills/dart-ing-drift-dynamic-tracing/SKILL.md) | Memory & Context, Efficiency & Optimization | 225 |
| [darwin-dynamic-agentically-rewriting](skills/darwin-dynamic-agentically-rewriting/SKILL.md) | Multi-Agent Systems, RAG & Retrieval, Evaluation & Benchmarking +4 | 167 |
| [data-centric-interpretability-llm-based-multi-agen](skills/data-centric-interpretability-llm-based-multi-agen/SKILL.md) | Multi-Agent Systems, Fine-tuning & Training, Agentic Systems +5 | 189 |
| [data-free-privacy-preserving-inversion-selective](skills/data-free-privacy-preserving-inversion-selective/SKILL.md) | Other | 150 |
| [datachef-cooking-up-optimal](skills/datachef-cooking-up-optimal/SKILL.md) | Fine-tuning & Training, Efficiency & Optimization, Data Processing | 207 |
| [datacross-unified-benchmark-agent](skills/datacross-unified-benchmark-agent/SKILL.md) | Code & Software Engineering, Evaluation & Benchmarking, Multimodal +2 | 203 |
| [david-vs-goliath-verifiable](skills/david-vs-goliath-verifiable/SKILL.md) | Security & Safety, Evaluation & Benchmarking, Agentic Systems | 165 |
| [davinci-agency-unlocking-long-horizon-agency](skills/davinci-agency-unlocking-long-horizon-agency/SKILL.md) | Code & Software Engineering | 181 |
| [davinci-dev-agent-native-mid-training-software](skills/davinci-dev-agent-native-mid-training-software/SKILL.md) | Code & Software Engineering, Fine-tuning & Training, Agentic Systems | 171 |
| [dcopilot-generative-ai-empowered-policy](skills/dcopilot-generative-ai-empowered-policy/SKILL.md) | Fine-tuning & Training, Prompt Engineering, Efficiency & Optimization | 297 |
| [debugging-code-world](skills/debugging-code-world/SKILL.md) | Code & Software Engineering | 171 |
| [decomposing-reasoning-efficiency](skills/decomposing-reasoning-efficiency/SKILL.md) | Reasoning & Chain-of-Thought, Efficiency & Optimization | 248 |
| [decoupled-reasoning-implicit-fact](skills/decoupled-reasoning-implicit-fact/SKILL.md) | Reasoning & Chain-of-Thought, Memory & Context, Efficiency & Optimization +1 | 164 |
| [decoupling-skeleton-flesh-multimodal](skills/decoupling-skeleton-flesh-multimodal/SKILL.md) | Reasoning & Chain-of-Thought, Multimodal, Data Processing +1 | 180 |
| [deep-researcher-sequential-plan](skills/deep-researcher-sequential-plan/SKILL.md) | RAG & Retrieval | 194 |
| [deep-search-hierarchical-meta-cognitive](skills/deep-search-hierarchical-meta-cognitive/SKILL.md) | RAG & Retrieval, Reasoning & Chain-of-Thought, Agentic Systems +1 | 188 |
| [deepasmr-llm-based-zero-shot-asmr](skills/deepasmr-llm-based-zero-shot-asmr/SKILL.md) | Multimodal, Prompt Engineering, Data Processing | 227 |
| [deepera-deep-evidence-reranking](skills/deepera-deep-evidence-reranking/SKILL.md) | RAG & Retrieval, Reasoning & Chain-of-Thought, Data Processing | 223 |
| [deepimagesearch-benchmarking-multimodal-agents](skills/deepimagesearch-benchmarking-multimodal-agents/SKILL.md) | RAG & Retrieval, Reasoning & Chain-of-Thought, Evaluation & Benchmarking +4 | 198 |
| [deepplanning-benchmarking-long-horizon-agentic](skills/deepplanning-benchmarking-long-horizon-agentic/SKILL.md) | Reasoning & Chain-of-Thought, Evaluation & Benchmarking, Agentic Systems +1 | 155 |
| [deepread-document-structure-aware-reasoning](skills/deepread-document-structure-aware-reasoning/SKILL.md) | Reasoning & Chain-of-Thought | 187 |
| [deltaevolve-accelerating-scientific-discovery](skills/deltaevolve-accelerating-scientific-discovery/SKILL.md) | RAG & Retrieval, Efficiency & Optimization | 187 |
| [dep-search-learning-dependency-aware-reasoning](skills/dep-search-learning-dependency-aware-reasoning/SKILL.md) | RAG & Retrieval, Reasoning & Chain-of-Thought, Memory & Context | 213 |
| [detecting-correcting-hallucinations-llm-generated](skills/detecting-correcting-hallucinations-llm-generated/SKILL.md) | Other | 260 |
| [devops-gym-benchmarking-ai-agents](skills/devops-gym-benchmarking-ai-agents/SKILL.md) | Code & Software Engineering, Evaluation & Benchmarking, Agentic Systems +1 | 170 |
| [dial-summer-structured-evaluation-framework](skills/dial-summer-structured-evaluation-framework/SKILL.md) | Evaluation & Benchmarking, Explainability | 244 |
| [diffa-2-practical-diffusion-general](skills/diffa-2-practical-diffusion-general/SKILL.md) | Fine-tuning & Training, Multimodal, Data Processing | 237 |
| [diffusion-lms-approximate-optimal](skills/diffusion-lms-approximate-optimal/SKILL.md) | RAG & Retrieval, Fine-tuning & Training | 170 |
| [diffusion-pretrained-dense-contextual-embeddings](skills/diffusion-pretrained-dense-contextual-embeddings/SKILL.md) | RAG & Retrieval, Fine-tuning & Training, Efficiency & Optimization +1 | 183 |
| [diffuspeech-silent-thought-spoken](skills/diffuspeech-silent-thought-spoken/SKILL.md) | Multimodal | 276 |
| [discovering-high-level-patterns](skills/discovering-high-level-patterns/SKILL.md) | Code & Software Engineering, Data Processing, NLP & Text | 190 |
| [discovering-process-outcome-credit-multi-step](skills/discovering-process-outcome-credit-multi-step/SKILL.md) | Other | 190 |
| [discoverllm-executing-intents-discovering](skills/discoverllm-executing-intents-discovering/SKILL.md) | Code & Software Engineering, Multimodal | 227 |
| [dispo-enhancing-training-efficiency](skills/dispo-enhancing-training-efficiency/SKILL.md) | Code & Software Engineering, Reasoning & Chain-of-Thought, Fine-tuning & Training +1 | 277 |
| [distilling-reasoning-graph-concept](skills/distilling-reasoning-graph-concept/SKILL.md) | Reasoning & Chain-of-Thought, Fine-tuning & Training, Efficiency & Optimization +2 | 170 |
| [diverge-diversity-enhanced-rag-open-ended](skills/diverge-diversity-enhanced-rag-open-ended/SKILL.md) | RAG & Retrieval, Memory & Context | 177 |
| [dllm-agent-see-farther](skills/dllm-agent-see-farther/SKILL.md) | Multi-Agent Systems, Agentic Systems, Efficiency & Optimization +2 | 163 |
| [dllm-searcher-adapting-diffusion-large](skills/dllm-searcher-adapting-diffusion-large/SKILL.md) | RAG & Retrieval, Reasoning & Chain-of-Thought, Fine-tuning & Training +2 | 256 |
| [do-reasoning-ask-questions](skills/do-reasoning-ask-questions/SKILL.md) | Reasoning & Chain-of-Thought, Agentic Systems, Efficiency & Optimization | 183 |
| [do-reasoning-enhance-embedding](skills/do-reasoning-enhance-embedding/SKILL.md) | Reasoning & Chain-of-Thought | 242 |
| [do-truly-benefit-longer](skills/do-truly-benefit-longer/SKILL.md) | RAG & Retrieval, Evaluation & Benchmarking, Efficiency & Optimization +2 | 252 |
| [do-vlms-have-moral](skills/do-vlms-have-moral/SKILL.md) | Security & Safety, Evaluation & Benchmarking, Multimodal +1 | 211 |
| [doc2spec-synthesizing-formal-programming](skills/doc2spec-synthesizing-formal-programming/SKILL.md) | Data Processing | 177 |
| [docksmith-scaling-reliable-coding](skills/docksmith-scaling-reliable-coding/SKILL.md) | Other | 201 |
| [domain-adaptation-synthetic-data-fine-tuning-germa](skills/domain-adaptation-synthetic-data-fine-tuning-germa/SKILL.md) | Fine-tuning & Training, Data Processing, Domain-Specific +1 | 193 |
| [domain-specific-knowledge-graphs-rag-enhanced](skills/domain-specific-knowledge-graphs-rag-enhanced/SKILL.md) | RAG & Retrieval | 217 |
| [dr-kernel-reinforcement-learning-done](skills/dr-kernel-reinforcement-learning-done/SKILL.md) | Efficiency & Optimization | 200 |
| [dr-mas-stable-reinforcement-learning](skills/dr-mas-stable-reinforcement-learning/SKILL.md) | Multi-Agent Systems, RAG & Retrieval, Reasoning & Chain-of-Thought +3 | 203 |
| [draincode-stealthy-energy-consumption](skills/draincode-stealthy-energy-consumption/SKILL.md) | RAG & Retrieval, Security & Safety, Code & Software Engineering +2 | 221 |
| [drpg-decompose-retrieve-plan](skills/drpg-decompose-retrieve-plan/SKILL.md) | RAG & Retrieval | 253 |
| [drugr-optimizing-molecular-drugs](skills/drugr-optimizing-molecular-drugs/SKILL.md) | Security & Safety, Reasoning & Chain-of-Thought, Fine-tuning & Training +4 | 187 |
| [duogen-general-purpose-interleaved](skills/duogen-general-purpose-interleaved/SKILL.md) | Reasoning & Chain-of-Thought, Fine-tuning & Training, Multimodal +1 | 209 |
| [dynamic-framework-collaborative-learning](skills/dynamic-framework-collaborative-learning/SKILL.md) | Other | 214 |
| [dynamic-long-context-reasoning](skills/dynamic-long-context-reasoning/SKILL.md) | Reasoning & Chain-of-Thought, Memory & Context | 210 |
| [dynamic-role-assignment-multi-agent](skills/dynamic-role-assignment-multi-agent/SKILL.md) | Multi-Agent Systems, Agentic Systems, Efficiency & Optimization | 182 |
| [dynaweb-model-based-reinforcement-learning](skills/dynaweb-model-based-reinforcement-learning/SKILL.md) | Fine-tuning & Training, Agentic Systems, Efficiency & Optimization +1 | 193 |
| [dziribot-rag-intelligent-conversational](skills/dziribot-rag-intelligent-conversational/SKILL.md) | RAG & Retrieval, Agentic Systems, Data Processing | 236 |
| [e2pl-prompt-learning-incomplete](skills/e2pl-prompt-learning-incomplete/SKILL.md) | Multimodal, Prompt Engineering, Efficiency & Optimization | 188 |
| [ecco-evidence-driven-causal-reasoning](skills/ecco-evidence-driven-causal-reasoning/SKILL.md) | Reasoning & Chain-of-Thought, Explainability | 205 |
| [ecg-agent-on-device-tool-calling-agent](skills/ecg-agent-on-device-tool-calling-agent/SKILL.md) | Agentic Systems, Efficiency & Optimization, Domain-Specific | 296 |
| [ecg-r1-protocol-guided-modality-agnostic-mllm](skills/ecg-r1-protocol-guided-modality-agnostic-mllm/SKILL.md) | Reasoning & Chain-of-Thought, Fine-tuning & Training, Data Processing +2 | 260 |
| [echo-open-research-platform](skills/echo-open-research-platform/SKILL.md) | RAG & Retrieval | 207 |
| [echoes-loop-diagnosing-risks](skills/echoes-loop-diagnosing-risks/SKILL.md) | Data Processing | 395 |
| [edge-optimized-vision-language-underground-infrast](skills/edge-optimized-vision-language-underground-infrast/SKILL.md) | Evaluation & Benchmarking, Multimodal, Efficiency & Optimization +2 | 483 |
| [effgen-enabling-small-language](skills/effgen-enabling-small-language/SKILL.md) | Memory & Context, Agentic Systems, Prompt Engineering +1 | 193 |
| [efficient-adaptable-detection-malicious](skills/efficient-adaptable-detection-malicious/SKILL.md) | Security & Safety, Efficiency & Optimization | 298 |
| [efficient-estimation-kernel-surrogate](skills/efficient-estimation-kernel-surrogate/SKILL.md) | Fine-tuning & Training, Efficiency & Optimization, Explainability | 199 |
| [efficient-table-retrieval-understanding](skills/efficient-table-retrieval-understanding/SKILL.md) | RAG & Retrieval, Efficiency & Optimization | 178 |
| [eft-cot-multi-agent-chain-of-thought-framework](skills/eft-cot-multi-agent-chain-of-thought-framework/SKILL.md) | Multi-Agent Systems, Reasoning & Chain-of-Thought, Agentic Systems +1 | 300 |
| [egss-entropy-guided-stepwise-scaling](skills/egss-entropy-guided-stepwise-scaling/SKILL.md) | Other | 164 |
| [eliciting-least-to-most-reasoning-phishing](skills/eliciting-least-to-most-reasoning-phishing/SKILL.md) | Reasoning & Chain-of-Thought, Evaluation & Benchmarking | 155 |
| [ema-policy-gradient-taming](skills/ema-policy-gradient-taming/SKILL.md) | RAG & Retrieval, Fine-tuning & Training, Agentic Systems | 228 |
| [embocoach-bench-benchmarking-ai-agents](skills/embocoach-bench-benchmarking-ai-agents/SKILL.md) | Evaluation & Benchmarking, Agentic Systems | 230 |
| [embodied-task-planning-graph-informed](skills/embodied-task-planning-graph-informed/SKILL.md) | Memory & Context, Knowledge Graphs, Agentic Systems +1 | 179 |
| [emoara-emotion-preserving-english-speech](skills/emoara-emotion-preserving-english-speech/SKILL.md) | Multimodal, Data Processing, NLP & Text | 217 |
| [emoshift-lightweight-activation-steering](skills/emoshift-lightweight-activation-steering/SKILL.md) | Fine-tuning & Training, Multimodal, Efficiency & Optimization | 227 |
| [emotion-llamav2-mmeverse-framework-benchmark](skills/emotion-llamav2-mmeverse-framework-benchmark/SKILL.md) | Multi-Agent Systems, Reasoning & Chain-of-Thought, Evaluation & Benchmarking +5 | 232 |
| [emotionthinker-prosody-aware-reinforcement-learnin](skills/emotionthinker-prosody-aware-reinforcement-learnin/SKILL.md) | Reasoning & Chain-of-Thought, Fine-tuning & Training, Multimodal +2 | 292 |
| [empirical-mcts-continuous-agent-evolution](skills/empirical-mcts-continuous-agent-evolution/SKILL.md) | RAG & Retrieval, Reasoning & Chain-of-Thought, Memory & Context +1 | 195 |
| [empowering-contrastive-federated-sequential](skills/empowering-contrastive-federated-sequential/SKILL.md) | Other | 169 |
| [energy-star-llm-enabled-software-engineering](skills/energy-star-llm-enabled-software-engineering/SKILL.md) | Other | 225 |
| [enhancing-mathematical-problem-solving](skills/enhancing-mathematical-problem-solving/SKILL.md) | Reasoning & Chain-of-Thought | 183 |
| [entworld-holistic-environment-benchmark](skills/entworld-holistic-environment-benchmark/SKILL.md) | Reasoning & Chain-of-Thought, Evaluation & Benchmarking, Agentic Systems | 158 |
| [environment-in-the-loop-rethinking-code-migration-](skills/environment-in-the-loop-rethinking-code-migration/SKILL.md) | Other | 162 |
| [epistemic-context-learning-building](skills/epistemic-context-learning-building/SKILL.md) | Multi-Agent Systems, Evaluation & Benchmarking, Agentic Systems +1 | 210 |
| [error-taxonomy-guided-prompt-optimization](skills/error-taxonomy-guided-prompt-optimization/SKILL.md) | Prompt Engineering, Efficiency & Optimization | 162 |
| [es-memeval-benchmarking-conversational-agents](skills/es-memeval-benchmarking-conversational-agents/SKILL.md) | Reasoning & Chain-of-Thought, Evaluation & Benchmarking, Memory & Context +2 | 226 |
| [eugens-unified-general-dense](skills/eugens-unified-general-dense/SKILL.md) | Other | 242 |
| [eurollm-22b-technical-report](skills/eurollm-22b-technical-report/SKILL.md) | Other | 202 |
| [evaluating-achieving-controllable-code](skills/evaluating-achieving-controllable-code/SKILL.md) | Evaluation & Benchmarking | 197 |
| [evaluating-enhancing-vulnerability-reasoning](skills/evaluating-enhancing-vulnerability-reasoning/SKILL.md) | Security & Safety, Code & Software Engineering, Reasoning & Chain-of-Thought +3 | 211 |
| [evaluating-kubernetes-performance-genai](skills/evaluating-kubernetes-performance-genai/SKILL.md) | Evaluation & Benchmarking, Efficiency & Optimization, Data Processing | 217 |
| [evaluating-retrievalaugmented-generation-variants](skills/evaluating-retrievalaugmented-generation-variants/SKILL.md) | RAG & Retrieval, Evaluation & Benchmarking | 223 |
| [evaluating-social-bias-rag](skills/evaluating-social-bias-rag/SKILL.md) | RAG & Retrieval, Evaluation & Benchmarking, Data Processing | 212 |
| [evaluating-they-not-know](skills/evaluating-they-not-know/SKILL.md) | Reasoning & Chain-of-Thought, Evaluation & Benchmarking, Efficiency & Optimization +1 | 185 |
| [evaluation-entity-matching-recommender](skills/evaluation-entity-matching-recommender/SKILL.md) | Evaluation & Benchmarking, Knowledge Graphs, Data Processing | 182 |
| [evaluation-legal-applications-challenges](skills/evaluation-legal-applications-challenges/SKILL.md) | Reasoning & Chain-of-Thought, Evaluation & Benchmarking, Data Processing +1 | 171 |
| [evaluation-oncotimia-system-supporting](skills/evaluation-oncotimia-system-supporting/SKILL.md) | RAG & Retrieval, Reasoning & Chain-of-Thought, Evaluation & Benchmarking +2 | 213 |
| [event-vstream-event-driven-real-time-understanding](skills/event-vstream-event-driven-real-time-understanding/SKILL.md) | Memory & Context, Multimodal, Efficiency & Optimization +1 | 254 |
| [eventcast-hybrid-demand-forecasting](skills/eventcast-hybrid-demand-forecasting/SKILL.md) | Reasoning & Chain-of-Thought, Data Processing | 228 |
| [evermembench-benchmarking-long-term-interactive](skills/evermembench-benchmarking-long-term-interactive/SKILL.md) | RAG & Retrieval, Evaluation & Benchmarking, Memory & Context +2 | 182 |
| [evocodebench-human-performance-benchmark-self-evol](skills/evocodebench-human-performance-benchmark-self-evol/SKILL.md) | Code & Software Engineering, Evaluation & Benchmarking, Memory & Context +3 | 174 |
| [evoconfig-self-evolving-multi-agent-systems](skills/evoconfig-self-evolving-multi-agent-systems/SKILL.md) | Multi-Agent Systems, Code & Software Engineering, Agentic Systems | 194 |
| [evolve-evolutionary-search-llm-based](skills/evolve-evolutionary-search-llm-based/SKILL.md) | RAG & Retrieval, Efficiency & Optimization | 174 |
| [evolving-tool-user-creator](skills/evolving-tool-user-creator/SKILL.md) | Reasoning & Chain-of-Thought, Fine-tuning & Training, Agentic Systems +2 | 181 |
| [ex-omni-enabling-3d-facial](skills/ex-omni-enabling-3d-facial/SKILL.md) | Multimodal, Data Processing | 204 |
| [experience-driven-multi-agent-systems-training-fre](skills/experience-driven-multi-agent-systems-training-fre/SKILL.md) | Multi-Agent Systems, RAG & Retrieval, Fine-tuning & Training +5 | 168 |
| [explainable-deepfake-detection-rl](skills/explainable-deepfake-detection-rl/SKILL.md) | Reasoning & Chain-of-Thought, Fine-tuning & Training, Multimodal +2 | 296 |
| [explicit-multi-head-attention-inter-head](skills/explicit-multi-head-attention-inter-head/SKILL.md) | Memory & Context | 217 |
| [exploring-reasoning-reward-agents](skills/exploring-reasoning-reward-agents/SKILL.md) | Reasoning & Chain-of-Thought, Agentic Systems | 219 |
| [extracting-recurring-vulnerabilities-black-box](skills/extracting-recurring-vulnerabilities-black-box/SKILL.md) | Security & Safety, Data Processing | 188 |
| [fademem-biologically-inspired-forgetting-agent](skills/fademem-biologically-inspired-forgetting-agent/SKILL.md) | Reasoning & Chain-of-Thought, Agentic Systems | 222 |
| [failure-aware-enhancements-code-generation](skills/failure-aware-enhancements-code-generation/SKILL.md) | Other | 191 |
| [farm-field-aware-resolution-intelligent](skills/farm-field-aware-resolution-intelligent/SKILL.md) | Multi-Agent Systems, RAG & Retrieval, Agentic Systems +1 | 187 |
| [fast-slow-training-multimodal-visual](skills/fast-slow-training-multimodal-visual/SKILL.md) | Fine-tuning & Training, Multimodal, Efficiency & Optimization | 230 |
| [fat-cat-document-driven-metacognitive-multi-agent](skills/fat-cat-document-driven-metacognitive-multi-agent/SKILL.md) | Multi-Agent Systems, Agentic Systems | 225 |
| [featurebench-benchmarking-agentic-coding](skills/featurebench-benchmarking-agentic-coding/SKILL.md) | Evaluation & Benchmarking, Agentic Systems, Data Processing | 178 |
| [fedkrso-communication-memory-federated](skills/fedkrso-communication-memory-federated/SKILL.md) | Fine-tuning & Training, Memory & Context, Efficiency & Optimization | 214 |
| [fimi-domain-specific-indian-finance](skills/fimi-domain-specific-indian-finance/SKILL.md) | Fine-tuning & Training, Agentic Systems, Data Processing +1 | 243 |
| [fin-rate-real-world-financial-analytics](skills/fin-rate-real-world-financial-analytics/SKILL.md) | RAG & Retrieval, Reasoning & Chain-of-Thought, Evaluation & Benchmarking +2 | 182 |
| [fine-r1-make-multi-modal-excel](skills/fine-r1-make-multi-modal-excel/SKILL.md) | Other | 178 |
| [fine-tuning-gpt-5-gpu-kernel](skills/fine-tuning-gpt-5-gpu-kernel/SKILL.md) | Evaluation & Benchmarking, Fine-tuning & Training, Efficiency & Optimization | 259 |
| [fit-defying-catastrophic-forgetting](skills/fit-defying-catastrophic-forgetting/SKILL.md) | Other | 174 |
| [flashvid-video-training-free-tree-based](skills/flashvid-video-training-free-tree-based/SKILL.md) | Fine-tuning & Training, Memory & Context, Multimodal +1 | 168 |
| [flexible-entropy-control-rlvr](skills/flexible-entropy-control-rlvr/SKILL.md) | Fine-tuning & Training | 205 |
| [flyaoc-evaluating-agentic-ontology](skills/flyaoc-evaluating-agentic-ontology/SKILL.md) | Multi-Agent Systems, RAG & Retrieval, Reasoning & Chain-of-Thought +4 | 184 |
| [fmbench-adaptive-output-formatting](skills/fmbench-adaptive-output-formatting/SKILL.md) | Other | 224 |
| [fnf-functional-network-fingerprint](skills/fnf-functional-network-fingerprint/SKILL.md) | Fine-tuning & Training | 215 |
| [focus-dllms-know-tame](skills/focus-dllms-know-tame/SKILL.md) | Evaluation & Benchmarking, Memory & Context, Efficiency & Optimization | 182 |
| [following-dragons-code-review-guided](skills/following-dragons-code-review-guided/SKILL.md) | RAG & Retrieval, Security & Safety, Code & Software Engineering +2 | 158 |
| [forest-chat-adapting-vision-language-agents](skills/forest-chat-adapting-vision-language-agents/SKILL.md) | Multi-Agent Systems, Reasoning & Chain-of-Thought, Multimodal +2 | 362 |
| [found-rl-foundation-model-enhanced-reinforcement](skills/found-rl-foundation-model-enhanced-reinforcement/SKILL.md) | Fine-tuning & Training, Multimodal, Efficiency & Optimization +1 | 264 |
| [fraudshield-knowledge-graph-empowered](skills/fraudshield-knowledge-graph-empowered/SKILL.md) | Prompt Engineering, Efficiency & Optimization, Data Processing | 269 |
| [from-assistant-double-agent](skills/from-assistant-double-agent/SKILL.md) | Security & Safety, Evaluation & Benchmarking, Memory & Context +2 | 230 |
| [from-assumptions-actions-turning](skills/from-assumptions-actions-turning/SKILL.md) | Multi-Agent Systems, Reasoning & Chain-of-Thought, Evaluation & Benchmarking +1 | 242 |
| [from-classification-ranking-enhancing](skills/from-classification-ranking-enhancing/SKILL.md) | Fine-tuning & Training, NLP & Text | 178 |
| [from-code-centric-concept-centric-teaching](skills/from-code-centric-concept-centric-teaching/SKILL.md) | Evaluation & Benchmarking, Prompt Engineering | 269 |
| [from-consistency-complementarity-aligned](skills/from-consistency-complementarity-aligned/SKILL.md) | Reasoning & Chain-of-Thought, Multimodal, Data Processing | 230 |
| [from-data-behavior-predicting](skills/from-data-behavior-predicting/SKILL.md) | Security & Safety, Fine-tuning & Training, Data Processing | 209 |
| [from-detection-prevention-explaining](skills/from-detection-prevention-explaining/SKILL.md) | Security & Safety, Code & Software Engineering, Explainability | 265 |
| [from-features-actions-explainability](skills/from-features-actions-explainability/SKILL.md) | Code & Software Engineering, Evaluation & Benchmarking, Agentic Systems +2 | 207 |
| [from-gameplay-traces-game](skills/from-gameplay-traces-game/SKILL.md) | Reasoning & Chain-of-Thought, Data Processing, Explainability +1 | 211 |
| [from-helpfulness-toxic-proactivity](skills/from-helpfulness-toxic-proactivity/SKILL.md) | Security & Safety, Evaluation & Benchmarking, Agentic Systems | 194 |
| [from-passive-metric-active](skills/from-passive-metric-active/SKILL.md) | RAG & Retrieval, Reasoning & Chain-of-Thought, Evaluation & Benchmarking +3 | 268 |
| [from-perception-action-spatial](skills/from-perception-action-spatial/SKILL.md) | Reasoning & Chain-of-Thought, Memory & Context, Agentic Systems +1 | 217 |
| [from-pragmas-partners-symbiotic](skills/from-pragmas-partners-symbiotic/SKILL.md) | RAG & Retrieval, Code & Software Engineering, Agentic Systems +2 | 186 |
| [from-prompt-response-goal-directed-systems](skills/from-prompt-response-goal-directed-systems/SKILL.md) | Multi-Agent Systems, Agentic Systems, Prompt Engineering +1 | 177 |
| [from-sparse-decisions-dense](skills/from-sparse-decisions-dense/SKILL.md) | Security & Safety, Reasoning & Chain-of-Thought, Evaluation & Benchmarking +3 | 261 |
| [from-task-solving-robust](skills/from-task-solving-robust/SKILL.md) | Reasoning & Chain-of-Thought, Agentic Systems, Data Processing | 199 |
| [from-utterance-vividity-training](skills/from-utterance-vividity-training/SKILL.md) | Fine-tuning & Training, Efficiency & Optimization, Data Processing +1 | 257 |
| [frost-filtering-reasoning-outliers](skills/frost-filtering-reasoning-outliers/SKILL.md) | Reasoning & Chain-of-Thought, Evaluation & Benchmarking, Memory & Context +1 | 252 |
| [fs-researcher-test-time-scaling-long-horizon](skills/fs-researcher-test-time-scaling-long-horizon/SKILL.md) | RAG & Retrieval | 221 |
| [fullstack-agent-enhancing-agentic-fullstack](skills/fullstack-agent-enhancing-agentic-fullstack/SKILL.md) | RAG & Retrieval, Code & Software Engineering, Agentic Systems +1 | 146 |
| [funny-or-persuasive-but](skills/funny-or-persuasive-but/SKILL.md) | Other | 178 |
| [funprm-function-as-step-process-reward](skills/funprm-function-as-step-process-reward/SKILL.md) | Code & Software Engineering, Reasoning & Chain-of-Thought, Evaluation & Benchmarking | 251 |
| [gamedevbench-evaluating-agentic-capabilities](skills/gamedevbench-evaluating-agentic-capabilities/SKILL.md) | Evaluation & Benchmarking, Multimodal, Agentic Systems | 187 |
| [gametalk-training-strategic-conversation](skills/gametalk-training-strategic-conversation/SKILL.md) | Multi-Agent Systems, Fine-tuning & Training, Agentic Systems +1 | 207 |
| [gamms-graph-based-adversarial](skills/gamms-graph-based-adversarial/SKILL.md) | Security & Safety, Knowledge Graphs | 305 |
| [gavel-rule-based-safety-activation](skills/gavel-rule-based-safety-activation/SKILL.md) | Security & Safety | 194 |
| [gdcnet-generative-discrepancy-comparison](skills/gdcnet-generative-discrepancy-comparison/SKILL.md) | Other | 180 |
| [gender-race-bias-consumer](skills/gender-race-bias-consumer/SKILL.md) | Data Processing | 255 |
| [generalizable-interpretable-rf-fingerprinting](skills/generalizable-interpretable-rf-fingerprinting/SKILL.md) | Prompt Engineering, Data Processing, Explainability | 168 |
| [generating-data-driven-reasoning-rubrics](skills/generating-data-driven-reasoning-rubrics/SKILL.md) | Reasoning & Chain-of-Thought, Evaluation & Benchmarking | 170 |
| [generative-ontology-structured-knowledge](skills/generative-ontology-structured-knowledge/SKILL.md) | Knowledge Graphs | 222 |
| [generative-visual-code-mobile](skills/generative-visual-code-mobile/SKILL.md) | Multimodal | 297 |
| [genius-generative-fluid-intelligence](skills/genius-generative-fluid-intelligence/SKILL.md) | Reasoning & Chain-of-Thought, Evaluation & Benchmarking, Memory & Context +2 | 247 |
| [geogr-generative-retrieval-framework](skills/geogr-generative-retrieval-framework/SKILL.md) | RAG & Retrieval | 171 |
| [gflowpo-generative-flow-network](skills/gflowpo-generative-flow-network/SKILL.md) | Evaluation & Benchmarking, Memory & Context, Prompt Engineering +1 | 171 |
| [gisa-benchmark-general-information-seeking](skills/gisa-benchmark-general-information-seeking/SKILL.md) | Evaluation & Benchmarking | 162 |
| [gradingattack-attacking-short-answer](skills/gradingattack-attacking-short-answer/SKILL.md) | Security & Safety, Evaluation & Benchmarking, Prompt Engineering | 243 |
| [graph-anchored-knowledge-indexing-retrieval-augmen](skills/graph-anchored-knowledge-indexing-retrieval-augmen/SKILL.md) | RAG & Retrieval, Knowledge Graphs, Data Processing | 221 |
| [graph-based-agent-memory-taxonomy](skills/graph-based-agent-memory-taxonomy/SKILL.md) | RAG & Retrieval, Memory & Context, Knowledge Graphs +2 | 279 |
| [graphagents-knowledge-graph-guided-agentic](skills/graphagents-knowledge-graph-guided-agentic/SKILL.md) | Multi-Agent Systems, RAG & Retrieval, Reasoning & Chain-of-Thought +3 | 185 |
| [graphdancer-training-explore-reason](skills/graphdancer-training-explore-reason/SKILL.md) | RAG & Retrieval, Reasoning & Chain-of-Thought, Fine-tuning & Training +2 | 243 |
| [graphseek-next-generation-graph-analytics](skills/graphseek-next-generation-graph-analytics/SKILL.md) | Reasoning & Chain-of-Thought, Knowledge Graphs, Agentic Systems +1 | 151 |
| [greprag-empirical-study-optimization](skills/greprag-empirical-study-optimization/SKILL.md) | RAG & Retrieval, Code & Software Engineering, Efficiency & Optimization | 190 |
| [grounding-generative-planners-verifiable](skills/grounding-generative-planners-verifiable/SKILL.md) | Other | 176 |
| [group-distributionally-robust-optimization-driven](skills/group-distributionally-robust-optimization-driven/SKILL.md) | Reasoning & Chain-of-Thought, Fine-tuning & Training, Prompt Engineering +1 | 205 |
| [guideai-real-time-personalized-learning](skills/guideai-real-time-personalized-learning/SKILL.md) | NLP & Text | 276 |
| [gutenocr-grounded-vision-language-front-end](skills/gutenocr-grounded-vision-language-front-end/SKILL.md) | Multimodal, Prompt Engineering, Data Processing | 210 |
| [haif-human-ai-integration-framework](skills/haif-human-ai-integration-framework/SKILL.md) | Evaluation & Benchmarking, Agentic Systems | 206 |
| [hallucination-resistant-security-planning](skills/hallucination-resistant-security-planning/SKILL.md) | Security & Safety, Agentic Systems | 259 |
| [halluverse-m3-multitask-multilingual-benchmark-hal](skills/halluverse-m3-multitask-multilingual-benchmark-hal/SKILL.md) | Evaluation & Benchmarking | 148 |
| [halt-hallucination-assessment-log-probs](skills/halt-hallucination-assessment-log-probs/SKILL.md) | Evaluation & Benchmarking | 315 |
| [harmoni-multimodal-personalization-multi-user](skills/harmoni-multimodal-personalization-multi-user/SKILL.md) | Memory & Context, Multimodal, Data Processing | 197 |
| [harnessing-precision-querying-retrieval-augmented](skills/harnessing-precision-querying-retrieval-augmented/SKILL.md) | RAG & Retrieval, Code & Software Engineering, Evaluation & Benchmarking +2 | 163 |
| [he-snr-uncovering-latent-logic](skills/he-snr-uncovering-latent-logic/SKILL.md) | Code & Software Engineering, Reasoning & Chain-of-Thought, Evaluation & Benchmarking +2 | 234 |
| [helios-hierarchical-graph-abstraction](skills/helios-hierarchical-graph-abstraction/SKILL.md) | Code & Software Engineering, Prompt Engineering | 204 |
| [helm-human-centered-evaluation-framework](skills/helm-human-centered-evaluation-framework/SKILL.md) | Evaluation & Benchmarking, Explainability | 244 |
| [hidden-licensing-risks-llmware](skills/hidden-licensing-risks-llmware/SKILL.md) | Multi-Agent Systems, Agentic Systems, Data Processing | 182 |
| [high-fidelity-textual-user](skills/high-fidelity-textual-user/SKILL.md) | RAG & Retrieval, Efficiency & Optimization | 227 |
| [history-guided-iterative-visual-reasoning](skills/history-guided-iterative-visual-reasoning/SKILL.md) | Reasoning & Chain-of-Thought, Multimodal | 170 |
| [how-decoder-only-perceive-users](skills/how-decoder-only-perceive-users/SKILL.md) | Fine-tuning & Training, Memory & Context, Data Processing +1 | 235 |
| [how-few-shot-demonstrations-affect](skills/how-few-shot-demonstrations-affect/SKILL.md) | Security & Safety, Prompt Engineering | 199 |
| [how-information-access-affect](skills/how-information-access-affect/SKILL.md) | Other | 194 |
| [how-much-reasoning-retrieval-augmented](skills/how-much-reasoning-retrieval-augmented/SKILL.md) | RAG & Retrieval, Reasoning & Chain-of-Thought, Evaluation & Benchmarking +2 | 178 |
| [how-personalized-memory-shape](skills/how-personalized-memory-shape/SKILL.md) | RAG & Retrieval, Reasoning & Chain-of-Thought, Memory & Context | 216 |
| [how-well-open-sourced](skills/how-well-open-sourced/SKILL.md) | Other | 232 |
| [hqp-sensitivity-aware-hybrid-quantization](skills/hqp-sensitivity-aware-hybrid-quantization/SKILL.md) | Fine-tuning & Training, Efficiency & Optimization | 189 |
| [hugrag-hierarchical-causal-knowledge](skills/hugrag-hierarchical-causal-knowledge/SKILL.md) | RAG & Retrieval, Reasoning & Chain-of-Thought, Knowledge Graphs +2 | 168 |
| [human-aligned-enhancement-programming-answers](skills/human-aligned-enhancement-programming-answers/SKILL.md) | Other | 258 |
| [humans-welcome-observe-first-look](skills/humans-welcome-observe-first-look/SKILL.md) | Security & Safety, Evaluation & Benchmarking, Agentic Systems | 191 |
| [hunt-instead-wait-evaluating](skills/hunt-instead-wait-evaluating/SKILL.md) | Evaluation & Benchmarking | 222 |
| [hybrid-supervised-llm-pipeline-actionable-suggesti](skills/hybrid-supervised-llm-pipeline-actionable-suggesti/SKILL.md) | Data Processing, NLP & Text | 193 |
| [hylra-hybrid-layer-reuse](skills/hylra-hybrid-layer-reuse/SKILL.md) | Memory & Context, Efficiency & Optimization | 190 |
| [hyperoffload-graph-driven-hierarchical-memory](skills/hyperoffload-graph-driven-hierarchical-memory/SKILL.md) | Code & Software Engineering, Fine-tuning & Training, Memory & Context +2 | 226 |
| [ic-eo-interpretable-code-based-assistant](skills/ic-eo-interpretable-code-based-assistant/SKILL.md) | Evaluation & Benchmarking, Multimodal, Agentic Systems +2 | 215 |
| [icl-evader-zero-query-black-box-evasion](skills/icl-evader-zero-query-black-box-evasion/SKILL.md) | Security & Safety, Prompt Engineering, Data Processing | 251 |
| [icon-intent-context-coupling-multi-turn](skills/icon-intent-context-coupling-multi-turn/SKILL.md) | Security & Safety, Evaluation & Benchmarking, Prompt Engineering | 243 |
| [ide-bench-evaluating-as-ide](skills/ide-bench-evaluating-as-ide/SKILL.md) | Code & Software Engineering, Evaluation & Benchmarking, Agentic Systems +1 | 149 |
| [identifying-adversary-tactics-techniques](skills/identifying-adversary-tactics-techniques/SKILL.md) | RAG & Retrieval, Code & Software Engineering, Reasoning & Chain-of-Thought +1 | 198 |
| [identifying-concurrency-bug-reports](skills/identifying-concurrency-bug-reports/SKILL.md) | Code & Software Engineering | 185 |
| [iesr-mcts-based-modular-reasoning](skills/iesr-mcts-based-modular-reasoning/SKILL.md) | RAG & Retrieval, Reasoning & Chain-of-Thought, Data Processing +1 | 242 |
| [improve-systems-user-logs](skills/improve-systems-user-logs/SKILL.md) | Fine-tuning & Training, Efficiency & Optimization, Data Processing | 201 |
| [improving-user-privacy-personalized](skills/improving-user-privacy-personalized/SKILL.md) | RAG & Retrieval, Evaluation & Benchmarking, Data Processing | 192 |
| [industrialized-deception-collateral-effects](skills/industrialized-deception-collateral-effects/SKILL.md) | Evaluation & Benchmarking, Data Processing | 216 |
| [infa-guard-mitigating-malicious-propagation](skills/infa-guard-mitigating-malicious-propagation/SKILL.md) | Security & Safety | 262 |
| [inficoevalchain-blockchain-based-decentralized-fra](skills/inficoevalchain-blockchain-based-decentralized-fra/SKILL.md) | Multi-Agent Systems, Evaluation & Benchmarking, Data Processing | 202 |
| [innovator-vl-multimodal-scientific-discovery](skills/innovator-vl-multimodal-scientific-discovery/SKILL.md) | Reasoning & Chain-of-Thought, Fine-tuning & Training, Multimodal +3 | 247 |
| [instructtime-time-series-classification-multimodal](skills/instructtime-time-series-classification-multimodal/SKILL.md) | Multimodal, Prompt Engineering, Data Processing | 165 |
| [integrating-fine-grained-audio-visual-evidence](skills/integrating-fine-grained-audio-visual-evidence/SKILL.md) | Multimodal | 284 |
| [internalizing-multi-agent-reasoning-accurate](skills/internalizing-multi-agent-reasoning-accurate/SKILL.md) | Multi-Agent Systems, RAG & Retrieval, Reasoning & Chain-of-Thought +4 | 174 |
| [internalizing-reasoning-discovery-replay](skills/internalizing-reasoning-discovery-replay/SKILL.md) | Reasoning & Chain-of-Thought, Fine-tuning & Training, Efficiency & Optimization +1 | 253 |
| [internet-agentic-ai-incentive-compatible](skills/internet-agentic-ai-incentive-compatible/SKILL.md) | Agentic Systems | 208 |
| [interpreting-agentic-systems-beyond](skills/interpreting-agentic-systems-beyond/SKILL.md) | Multi-Agent Systems, Code & Software Engineering, Evaluation & Benchmarking +3 | 329 |
| [interpreting-controlling-behavior-constitutions](skills/interpreting-controlling-behavior-constitutions/SKILL.md) | Multimodal, Prompt Engineering, Explainability | 182 |
| [interpreting-controlling-reasoning-integrated](skills/interpreting-controlling-reasoning-integrated/SKILL.md) | Reasoning & Chain-of-Thought, Data Processing, Explainability | 209 |
| [intraslice-high-performance-structural-pruning](skills/intraslice-high-performance-structural-pruning/SKILL.md) | Memory & Context, Efficiency & Optimization | 204 |
| [isd-agent-bench-comprehensive-benchmark-evaluating](skills/isd-agent-bench-comprehensive-benchmark-evaluating/SKILL.md) | Evaluation & Benchmarking, Fine-tuning & Training, Agentic Systems +1 | 210 |
| [issueguard-real-time-secret-leak](skills/issueguard-real-time-secret-leak/SKILL.md) | Data Processing | 268 |
| [iterative-refinement-improves-compositional](skills/iterative-refinement-improves-compositional/SKILL.md) | Multimodal, Agentic Systems, Prompt Engineering +1 | 199 |
| [jacobian-scopes-token-level-causal](skills/jacobian-scopes-token-level-causal/SKILL.md) | Code & Software Engineering, Explainability | 223 |
| [jade-bridging-strategic-operational-gap](skills/jade-bridging-strategic-operational-gap/SKILL.md) | RAG & Retrieval, Agentic Systems, Efficiency & Optimization +2 | 248 |
| [jaf-judge-agent-forest](skills/jaf-judge-agent-forest/SKILL.md) | Agentic Systems | 213 |
| [jailbreaking-calibration](skills/jailbreaking-calibration/SKILL.md) | Security & Safety | 211 |
| [jailbreaks-vision-multimodal-reasoning](skills/jailbreaks-vision-multimodal-reasoning/SKILL.md) | Security & Safety, Reasoning & Chain-of-Thought, Multimodal | 250 |
| [jetformer-scalable-transformer-jet](skills/jetformer-scalable-transformer-jet/SKILL.md) | RAG & Retrieval, Efficiency & Optimization | 167 |
| [jobresqa-benchmark-machine-reading](skills/jobresqa-benchmark-machine-reading/SKILL.md) | Reasoning & Chain-of-Thought, Evaluation & Benchmarking, Data Processing +1 | 152 |
| [joint-continual-learning-local](skills/joint-continual-learning-local/SKILL.md) | Fine-tuning & Training, Efficiency & Optimization, Data Processing | 270 |
| [just-ask-curious-code](skills/just-ask-curious-code/SKILL.md) | Other | 224 |
| [just-in-time-reinforcement-learning-continual](skills/just-in-time-reinforcement-learning-continual/SKILL.md) | Evaluation & Benchmarking, Fine-tuning & Training, Memory & Context +3 | 203 |
| [kg-craft-knowledge-graph-based-contrastive](skills/kg-craft-knowledge-graph-based-contrastive/SKILL.md) | Knowledge Graphs | 197 |
| [kid-knowledge-injected-dual-head-learning](skills/kid-knowledge-injected-dual-head-learning/SKILL.md) | Security & Safety, Reasoning & Chain-of-Thought, Multimodal +1 | 185 |
| [knowledge-graphs-implicit-reward](skills/knowledge-graphs-implicit-reward/SKILL.md) | Reasoning & Chain-of-Thought, Fine-tuning & Training, Knowledge Graphs +1 | 168 |
| [knowledge-restoration-driven-prompt-optimization](skills/knowledge-restoration-driven-prompt-optimization/SKILL.md) | Prompt Engineering, Efficiency & Optimization | 211 |
| [koral-knowledge-graph-guided](skills/koral-knowledge-graph-guided/SKILL.md) | RAG & Retrieval, Reasoning & Chain-of-Thought, Knowledge Graphs +1 | 247 |
| [krone-hierarchical-modular-log](skills/krone-hierarchical-modular-log/SKILL.md) | Reasoning & Chain-of-Thought | 240 |
| [kv-core-benchmarking-data-dependent-low-rank](skills/kv-core-benchmarking-data-dependent-low-rank/SKILL.md) | Evaluation & Benchmarking, Memory & Context, Efficiency & Optimization | 195 |
| [large-geolocation-extraction-humanitarian](skills/large-geolocation-extraction-humanitarian/SKILL.md) | Agentic Systems, Prompt Engineering, Data Processing +1 | 213 |
| [large-model-powered-evolutionary-code](skills/large-model-powered-evolutionary-code/SKILL.md) | RAG & Retrieval, Memory & Context, Efficiency & Optimization | 186 |
| [large-reasoning-failures](skills/large-reasoning-failures/SKILL.md) | RAG & Retrieval, Code & Software Engineering, Reasoning & Chain-of-Thought +1 | 218 |
| [large-scale-multidimensional-knowledge-profiling](skills/large-scale-multidimensional-knowledge-profiling/SKILL.md) | RAG & Retrieval, Reasoning & Chain-of-Thought, Data Processing | 246 |
| [lata-tool-llm-assisted-translation](skills/lata-tool-llm-assisted-translation/SKILL.md) | NLP & Text | 232 |
| [latent-chain-of-thought-as-planning](skills/latent-chain-of-thought-as-planning/SKILL.md) | RAG & Retrieval, Reasoning & Chain-of-Thought, Agentic Systems | 200 |
| [latentchem-textual-cot-latent](skills/latentchem-textual-cot-latent/SKILL.md) | Reasoning & Chain-of-Thought, Agentic Systems, Efficiency & Optimization +1 | 189 |
| [layer-wise-lora-fine-tuning-similarity](skills/layer-wise-lora-fine-tuning-similarity/SKILL.md) | Evaluation & Benchmarking, Fine-tuning & Training, Efficiency & Optimization | 221 |
| [learning-compose-cross-domain-agentic](skills/learning-compose-cross-domain-agentic/SKILL.md) | Multi-Agent Systems, Reasoning & Chain-of-Thought, Agentic Systems +1 | 159 |
| [learning-decentralized-collaboration-multi-agent](skills/learning-decentralized-collaboration-multi-agent/SKILL.md) | Multi-Agent Systems, Evaluation & Benchmarking, Agentic Systems +1 | 218 |
| [learning-decode-against-compositional](skills/learning-decode-against-compositional/SKILL.md) | Reasoning & Chain-of-Thought, Evaluation & Benchmarking, Multimodal +2 | 284 |
| [learning-irrecoverable-error-localized-policy](skills/learning-irrecoverable-error-localized-policy/SKILL.md) | RAG & Retrieval, Code & Software Engineering, Reasoning & Chain-of-Thought +2 | 175 |
| [learning-rate-matters-vanilla](skills/learning-rate-matters-vanilla/SKILL.md) | RAG & Retrieval, Fine-tuning & Training, Efficiency & Optimization | 203 |
| [learning-reason-faithfully-step-level](skills/learning-reason-faithfully-step-level/SKILL.md) | Reasoning & Chain-of-Thought | 173 |
| [lec-kg-llm-embedding-collaborative-framework](skills/lec-kg-llm-embedding-collaborative-framework/SKILL.md) | Reasoning & Chain-of-Thought, Knowledge Graphs, Data Processing | 186 |
| [legalmalr-multi-agent-query-understanding](skills/legalmalr-multi-agent-query-understanding/SKILL.md) | Multi-Agent Systems, RAG & Retrieval, Agentic Systems +2 | 168 |
| [legalone-family-foundation-reliable](skills/legalone-family-foundation-reliable/SKILL.md) | Reasoning & Chain-of-Thought, Fine-tuning & Training, Agentic Systems +3 | 259 |
| [lemon-agent-technical-report](skills/lemon-agent-technical-report/SKILL.md) | Multi-Agent Systems, Memory & Context, Agentic Systems +2 | 186 |
| [lemur-corpus-robust-fine-tuning](skills/lemur-corpus-robust-fine-tuning/SKILL.md) | RAG & Retrieval, Evaluation & Benchmarking, Fine-tuning & Training +2 | 234 |
| [less-enough-synthesizing-diverse](skills/less-enough-synthesizing-diverse/SKILL.md) | RAG & Retrieval, Fine-tuning & Training, Efficiency & Optimization +1 | 181 |
| [less-finetuning-retrieval-rethinking](skills/less-finetuning-retrieval-rethinking/SKILL.md) | RAG & Retrieval | 269 |
| [less-noise-more-voice](skills/less-noise-more-voice/SKILL.md) | Reasoning & Chain-of-Thought, Multimodal, Prompt Engineering | 236 |
| [leveraging-data-say-no](skills/leveraging-data-say-no/SKILL.md) | RAG & Retrieval, Evaluation & Benchmarking, Memory & Context +2 | 194 |
| [leveraging-turkish-skill-extraction](skills/leveraging-turkish-skill-extraction/SKILL.md) | RAG & Retrieval, Reasoning & Chain-of-Thought, Prompt Engineering +2 | 198 |
| [lhaw-controllable-underspecification-long-horizon](skills/lhaw-controllable-underspecification-long-horizon/SKILL.md) | Agentic Systems, Prompt Engineering | 170 |
| [linear-merging-unlocks-simple](skills/linear-merging-unlocks-simple/SKILL.md) | RAG & Retrieval, Fine-tuning & Training, Multimodal +1 | 168 |
| [linglanmidian-systematic-evaluation-tcm](skills/linglanmidian-systematic-evaluation-tcm/SKILL.md) | Evaluation & Benchmarking, Data Processing, Domain-Specific | 245 |
| [lingua-safetybench-benchmark-safety-evaluation-mul](skills/lingua-safetybench-benchmark-safety-evaluation-mul/SKILL.md) | Security & Safety, Evaluation & Benchmarking, Multimodal +1 | 160 |
| [linguamap-which-layers-speak](skills/linguamap-which-layers-speak/SKILL.md) | Fine-tuning & Training | 230 |
| [linguistagent-a-reflective-multimodel](skills/linguistagent-a-reflective-multimodel/SKILL.md) | Agentic Systems | 241 |
| [lingxidiagbench-multi-agent-framework-benchmarking](skills/lingxidiagbench-multi-agent-framework-benchmarking/SKILL.md) | Multi-Agent Systems, Reasoning & Chain-of-Thought, Evaluation & Benchmarking +1 | 216 |
| [live-evo-online-evolution-agentic](skills/live-evo-online-evolution-agentic/SKILL.md) | RAG & Retrieval, Memory & Context, Agentic Systems +1 | 204 |
| [livemedbench-contamination-free-medical-benchmark](skills/livemedbench-contamination-free-medical-benchmark/SKILL.md) | Multi-Agent Systems, Evaluation & Benchmarking, Agentic Systems +2 | 296 |
| [livibench-omnimodal-benchmark-interactive](skills/livibench-omnimodal-benchmark-interactive/SKILL.md) | Multi-Agent Systems, RAG & Retrieval, Evaluation & Benchmarking +3 | 238 |
| [llama-31-foundationai-securityllm-reasoning-8b-tec](skills/llama-31-foundationai-securityllm-reasoning-8b-tec/SKILL.md) | Security & Safety, Reasoning & Chain-of-Thought | 251 |
| [llamea-sage-guiding-automated-algorithm](skills/llamea-sage-guiding-automated-algorithm/SKILL.md) | Code & Software Engineering, Efficiency & Optimization, Data Processing +1 | 226 |
| [llm-assisted-logic-rule-learning](skills/llm-assisted-logic-rule-learning/SKILL.md) | Reasoning & Chain-of-Thought, Efficiency & Optimization, Explainability | 181 |
| [llm-autodp-automatic-data-processing](skills/llm-autodp-automatic-data-processing/SKILL.md) | RAG & Retrieval, Fine-tuning & Training, Agentic Systems +2 | 168 |
| [llm-based-sql-generation-prompting](skills/llm-based-sql-generation-prompting/SKILL.md) | Prompt Engineering, Data Processing | 174 |
| [llm-enhanced-reinforcement-learning-long-term](skills/llm-enhanced-reinforcement-learning-long-term/SKILL.md) | Reasoning & Chain-of-Thought, Fine-tuning & Training, Agentic Systems +2 | 247 |
| [llm-fsm-scaling-finite-state-reasoning](skills/llm-fsm-scaling-finite-state-reasoning/SKILL.md) | Reasoning & Chain-of-Thought, NLP & Text | 265 |
| [llm-in-sandbox-elicits-general-agentic](skills/llm-in-sandbox-elicits-general-agentic/SKILL.md) | Memory & Context, Agentic Systems | 198 |
| [llm-not-all-you](skills/llm-not-all-you/SKILL.md) | Fine-tuning & Training, Multimodal, Prompt Engineering +3 | 187 |
| [llm-prompt-evaluation-educational](skills/llm-prompt-evaluation-educational/SKILL.md) | Evaluation & Benchmarking, Prompt Engineering, Efficiency & Optimization | 223 |
| [llms-as-high-dimensional-nonlinear](skills/llms-as-high-dimensional-nonlinear/SKILL.md) | Code & Software Engineering, Reasoning & Chain-of-Thought, Fine-tuning & Training +2 | 190 |
| [llms-as-orchestrators-constraint-compliant](skills/llms-as-orchestrators-constraint-compliant/SKILL.md) | Multi-Agent Systems | 186 |
| [llms-encode-failures-predicting](skills/llms-encode-failures-predicting/SKILL.md) | Efficiency & Optimization | 192 |
| [lmmrec-llm-driven-motivation-aware-multimodal](skills/lmmrec-llm-driven-motivation-aware-multimodal/SKILL.md) | Reasoning & Chain-of-Thought, Multimodal, Prompt Engineering +1 | 217 |
| [loca-bench-benchmarking-agents-under](skills/loca-bench-benchmarking-agents-under/SKILL.md) | Evaluation & Benchmarking, Memory & Context, Agentic Systems | 169 |
| [localv-exploiting-information-locality](skills/localv-exploiting-information-locality/SKILL.md) | Multi-Agent Systems, Code & Software Engineering, Agentic Systems | 138 |
| [locomo-plus-beyond-factual-cognitive-memory](skills/locomo-plus-beyond-factual-cognitive-memory/SKILL.md) | RAG & Retrieval, Evaluation & Benchmarking, Memory & Context +2 | 216 |
| [logicscore-fine-grained-logic-evaluation](skills/logicscore-fine-grained-logic-evaluation/SKILL.md) | Reasoning & Chain-of-Thought, Evaluation & Benchmarking, Data Processing | 185 |
| [logsieve-task-aware-ci-log](skills/logsieve-task-aware-ci-log/SKILL.md) | Code & Software Engineering, NLP & Text | 167 |
| [longcat-flash-thinking-2601-technical-report](skills/longcat-flash-thinking-2601-technical-report/SKILL.md) | Multi-Agent Systems, Reasoning & Chain-of-Thought, Agentic Systems +1 | 311 |
| [lost-translation-comparative-study](skills/lost-translation-comparative-study/SKILL.md) | Security & Safety, Evaluation & Benchmarking, NLP & Text | 176 |
| [lps-bench-benchmarking-safety-awareness](skills/lps-bench-benchmarking-safety-awareness/SKILL.md) | Security & Safety, Evaluation & Benchmarking | 191 |
| [lycheedecode-accelerating-long-context-inference](skills/lycheedecode-accelerating-long-context-inference/SKILL.md) | RAG & Retrieval, Memory & Context, Efficiency & Optimization | 179 |
| [mad-modality-adaptive-decoding-mitigating](skills/mad-modality-adaptive-decoding-mitigating/SKILL.md) | Evaluation & Benchmarking, Multimodal, Data Processing | 227 |
| [made-benchmark-environments-closed-loop](skills/made-benchmark-environments-closed-loop/SKILL.md) | RAG & Retrieval, Evaluation & Benchmarking, Agentic Systems +2 | 144 |
| [magellan-autonomous-discovery-compiler](skills/magellan-autonomous-discovery-compiler/SKILL.md) | RAG & Retrieval, Code & Software Engineering, Reasoning & Chain-of-Thought +2 | 158 |
| [malicious-agent-skills-wild](skills/malicious-agent-skills-wild/SKILL.md) | Security & Safety, Agentic Systems | 264 |
| [malicious-repurposing-open-science](skills/malicious-repurposing-open-science/SKILL.md) | Security & Safety | 212 |
| [malloc-benchmarking-memory-aware-long](skills/malloc-benchmarking-memory-aware-long/SKILL.md) | Evaluation & Benchmarking, Memory & Context, Efficiency & Optimization | 248 |
| [marble-multi-agent-reasoning-bioinformatics](skills/marble-multi-agent-reasoning-bioinformatics/SKILL.md) | Multi-Agent Systems, Reasoning & Chain-of-Thought, Memory & Context +3 | 206 |
| [markovscale-optimal-sequential-scaling](skills/markovscale-optimal-sequential-scaling/SKILL.md) | Efficiency & Optimization, Data Processing | 193 |
| [marti-mars2-scaling-multi-agent-self-search-reinfo](skills/marti-mars2-scaling-multi-agent-self-search-reinfo/SKILL.md) | Multi-Agent Systems, RAG & Retrieval, Code & Software Engineering +1 | 204 |
| [martingale-foresight-sampling-principled](skills/martingale-foresight-sampling-principled/SKILL.md) | RAG & Retrieval, Reasoning & Chain-of-Thought, Efficiency & Optimization | 220 |
| [mas-prove-understanding-process-verification](skills/mas-prove-understanding-process-verification/SKILL.md) | Multi-Agent Systems, Reasoning & Chain-of-Thought, Evaluation & Benchmarking +3 | 237 |
| [masalbench-benchmark-contextual-cross-cultural](skills/masalbench-benchmark-contextual-cross-cultural/SKILL.md) | Evaluation & Benchmarking, Data Processing | 186 |
| [mascot-multi-agent-socio-collaborative-companion](skills/mascot-multi-agent-socio-collaborative-companion/SKILL.md) | Multi-Agent Systems, Reasoning & Chain-of-Thought, Agentic Systems +1 | 244 |
| [massive-sound-embedding-benchmark](skills/massive-sound-embedding-benchmark/SKILL.md) | Evaluation & Benchmarking | 236 |
| [mata-multiagent-framework-for](skills/mata-multiagent-framework-for/SKILL.md) | Multi-Agent Systems, Reasoning & Chain-of-Thought, Agentic Systems +3 | 167 |
| [mata-trainable-hierarchical-automaton](skills/mata-trainable-hierarchical-automaton/SKILL.md) | Multi-Agent Systems, Reasoning & Chain-of-Thought, Memory & Context +3 | 303 |
| [mathliblemma-folklore-lemma-generation](skills/mathliblemma-folklore-lemma-generation/SKILL.md) | Multi-Agent Systems, Reasoning & Chain-of-Thought, Agentic Systems | 181 |
| [mcp-atlas-large-scale-benchmark-tool-use](skills/mcp-atlas-large-scale-benchmark-tool-use/SKILL.md) | Evaluation & Benchmarking, Agentic Systems | 210 |
| [mdl-unified-multi-distribution-learner](skills/mdl-unified-multi-distribution-learner/SKILL.md) | Memory & Context, NLP & Text | 177 |
| [medbeads-agent-native-immutable-data](skills/medbeads-agent-native-immutable-data/SKILL.md) | RAG & Retrieval, Agentic Systems, Data Processing +2 | 182 |
| [medmo-grounding-understanding-multimodal](skills/medmo-grounding-understanding-multimodal/SKILL.md) | Reasoning & Chain-of-Thought, Fine-tuning & Training, Multimodal +2 | 313 |
| [medsam-agent-empowering-interactive-medical](skills/medsam-agent-empowering-interactive-medical/SKILL.md) | Agentic Systems, Domain-Specific | 223 |
| [medspeak-knowledge-graph-aided-asr](skills/medspeak-knowledge-graph-aided-asr/SKILL.md) | RAG & Retrieval, Reasoning & Chain-of-Thought, Multimodal +3 | 262 |
| [medverse-reliable-medical-reasoning](skills/medverse-reliable-medical-reasoning/SKILL.md) | Reasoning & Chain-of-Thought, Domain-Specific | 216 |
| [meetbench-xl-calibrated-multidimensional-evaluation](skills/meetbench-xl-calibrated-multidimensional-evaluation/SKILL.md) | Evaluation & Benchmarking | 191 |
| [meki-memory-based-expert-knowledge](skills/meki-memory-based-expert-knowledge/SKILL.md) | Memory & Context | 265 |
| [memadapter-fast-alignment-across](skills/memadapter-fast-alignment-across/SKILL.md) | RAG & Retrieval, Evaluation & Benchmarking, Fine-tuning & Training +3 | 166 |
| [memcast-memory-driven-time-series](skills/memcast-memory-driven-time-series/SKILL.md) | RAG & Retrieval, Reasoning & Chain-of-Thought, Memory & Context +1 | 196 |
| [mempot-defending-against-memory](skills/mempot-defending-against-memory/SKILL.md) | Memory & Context | 246 |
| [menaspeechbank-reference-voice-bank](skills/menaspeechbank-reference-voice-bank/SKILL.md) | Multimodal | 177 |
| [menvagent-scalable-polyglot-environment](skills/menvagent-scalable-polyglot-environment/SKILL.md) | Multi-Agent Systems, Agentic Systems | 188 |
| [merlin-discovery-engine-photonic](skills/merlin-discovery-engine-photonic/SKILL.md) | Evaluation & Benchmarking, Fine-tuning & Training | 248 |
| [mermaid-memory-enhanced-retrieval-reasoning](skills/mermaid-memory-enhanced-retrieval-reasoning/SKILL.md) | Multi-Agent Systems, RAG & Retrieval, Reasoning & Chain-of-Thought +4 | 189 |
| [metagen-self-evolving-roles-topologies](skills/metagen-self-evolving-roles-topologies/SKILL.md) | Agentic Systems | 189 |
| [metaphorstar-image-metaphor-understanding](skills/metaphorstar-image-metaphor-understanding/SKILL.md) | Reasoning & Chain-of-Thought, Multimodal, Explainability | 197 |
| [mhdash-online-platform-benchmarking](skills/mhdash-online-platform-benchmarking/SKILL.md) | Security & Safety, Evaluation & Benchmarking, Data Processing | 285 |
| [mind-ambiguity-aleatoric-uncertainty](skills/mind-ambiguity-aleatoric-uncertainty/SKILL.md) | Security & Safety, Reasoning & Chain-of-Thought, Data Processing +1 | 225 |
| [mirror-multi-agent-framework-iterative](skills/mirror-multi-agent-framework-iterative/SKILL.md) | Multi-Agent Systems, RAG & Retrieval, Reasoning & Chain-of-Thought +4 | 166 |
| [misconception-diagnosis-student-tutor-dialogue](skills/misconception-diagnosis-student-tutor-dialogue/SKILL.md) | Other | 218 |
| [mitigating-conversational-inertia-multi-turn](skills/mitigating-conversational-inertia-multi-turn/SKILL.md) | Agentic Systems, Prompt Engineering | 236 |
| [mixing-expert-knowledge-bring](skills/mixing-expert-knowledge-bring/SKILL.md) | Reasoning & Chain-of-Thought, Fine-tuning & Training, Data Processing | 199 |
| [mmr-bench-comprehensive-benchmark-multimodal](skills/mmr-bench-comprehensive-benchmark-multimodal/SKILL.md) | Evaluation & Benchmarking, Multimodal, Efficiency & Optimization | 175 |
| [mmts-bench-comprehensive-benchmark-time](skills/mmts-bench-comprehensive-benchmark-time/SKILL.md) | Reasoning & Chain-of-Thought, Evaluation & Benchmarking, Data Processing | 193 |
| [mock-worlds-real-skills](skills/mock-worlds-real-skills/SKILL.md) | Other | 201 |
| [moco-one-stop-shop-collaboration](skills/moco-one-stop-shop-collaboration/SKILL.md) | Multi-Agent Systems, Agentic Systems, Data Processing | 225 |
| [modality-gap-driven-subspace-alignment](skills/modality-gap-driven-subspace-alignment/SKILL.md) | RAG & Retrieval, Evaluation & Benchmarking, Fine-tuning & Training +1 | 231 |
| [modular-multi-task-learning-chemical](skills/modular-multi-task-learning-chemical/SKILL.md) | Other | 209 |
| [monotonicity-as-architectural-bias](skills/monotonicity-as-architectural-bias/SKILL.md) | Other | 204 |
| [monte-carlo-tree-search](skills/monte-carlo-tree-search/SKILL.md) | RAG & Retrieval | 186 |
| [more-code-less-reuse](skills/more-code-less-reuse/SKILL.md) | Code & Software Engineering | 187 |
| [more-than-quick-glance](skills/more-than-quick-glance/SKILL.md) | Memory & Context, Efficiency & Optimization, Data Processing | 256 |
| [mpib-benchmark-medical-prompt](skills/mpib-benchmark-medical-prompt/SKILL.md) | RAG & Retrieval, Security & Safety, Code & Software Engineering +4 | 177 |
| [mrag-benchmarking-retrieval-augmented-generation](skills/mrag-benchmarking-retrieval-augmented-generation/SKILL.md) | RAG & Retrieval, Evaluation & Benchmarking, Prompt Engineering +3 | 183 |
| [muco-multi-turn-contrastive-learning](skills/muco-multi-turn-contrastive-learning/SKILL.md) | RAG & Retrieval, Fine-tuning & Training, Multimodal +1 | 260 |
| [multi-agent-causal-reasoning-system](skills/multi-agent-causal-reasoning-system/SKILL.md) | Multi-Agent Systems, Reasoning & Chain-of-Thought, Agentic Systems +2 | 225 |
| [multi-agent-collaborative-intrusion-detection](skills/multi-agent-collaborative-intrusion-detection/SKILL.md) | Multi-Agent Systems, Agentic Systems, Data Processing | 309 |
| [multi-agent-constraint-factorization-reveals](skills/multi-agent-constraint-factorization-reveals/SKILL.md) | Multi-Agent Systems, Agentic Systems, Data Processing | 157 |
| [multi-agent-end-to-end-vulnerability-management](skills/multi-agent-end-to-end-vulnerability-management/SKILL.md) | Multi-Agent Systems, Security & Safety, Agentic Systems +1 | 196 |
| [multi-agent-teams-hold-experts](skills/multi-agent-teams-hold-experts/SKILL.md) | Multi-Agent Systems, RAG & Retrieval, Agentic Systems +1 | 151 |
| [multi-agentic-ai-fairness-aware-accelerated](skills/multi-agentic-ai-fairness-aware-accelerated/SKILL.md) | Multi-Agent Systems, Agentic Systems, Prompt Engineering +1 | 200 |
| [multi-field-tool-retrieval](skills/multi-field-tool-retrieval/SKILL.md) | RAG & Retrieval, Agentic Systems, Efficiency & Optimization | 284 |
| [multi-targeted-graph-backdoor-attack](skills/multi-targeted-graph-backdoor-attack/SKILL.md) | Security & Safety, Evaluation & Benchmarking, Knowledge Graphs +1 | 193 |
| [multi-task-code-data-mix](skills/multi-task-code-data-mix/SKILL.md) | Other | 214 |
| [multi-task-grpo-reliable-reasoning](skills/multi-task-grpo-reliable-reasoning/SKILL.md) | Reasoning & Chain-of-Thought, Fine-tuning & Training | 228 |
| [multimodal-fine-tuning-synthetic-captions](skills/multimodal-fine-tuning-synthetic-captions/SKILL.md) | Fine-tuning & Training, Multimodal, Prompt Engineering +1 | 201 |
| [multimodal-learning-arcing-detection](skills/multimodal-learning-arcing-detection/SKILL.md) | Fine-tuning & Training, Multimodal | 215 |
| [multimodal-multi-agent-ransomware-analysis](skills/multimodal-multi-agent-ransomware-analysis/SKILL.md) | Multi-Agent Systems, Security & Safety, Multimodal +2 | 261 |
| [multivis-agent-multi-agent-framework-logic](skills/multivis-agent-multi-agent-framework-logic/SKILL.md) | Multi-Agent Systems, Reasoning & Chain-of-Thought, Multimodal +2 | 193 |
| [mulvul-retrieval-augmented-multi-agent-code](skills/mulvul-retrieval-augmented-multi-agent-code/SKILL.md) | Multi-Agent Systems, RAG & Retrieval, Security & Safety +2 | 204 |
| [naamse-framework-evolutionary-security](skills/naamse-framework-evolutionary-security/SKILL.md) | Security & Safety, Evaluation & Benchmarking, Agentic Systems +1 | 201 |
| [nag-unified-native-architecture](skills/nag-unified-native-architecture/SKILL.md) | Reasoning & Chain-of-Thought, Memory & Context, Knowledge Graphs | 204 |
| [natural-instructions-scene-responsive-human-in-the](skills/natural-instructions-scene-responsive-human-in-the/SKILL.md) | Other | 372 |
| [neural-theorem-proving-verification](skills/neural-theorem-proving-verification/SKILL.md) | Reasoning & Chain-of-Thought, NLP & Text | 202 |
| [neurofilter-privacy-guardrails-conversational](skills/neurofilter-privacy-guardrails-conversational/SKILL.md) | Security & Safety | 288 |
| [next-gen-captchas-leveraging-cognitive](skills/next-gen-captchas-leveraging-cognitive/SKILL.md) | RAG & Retrieval, Agentic Systems, Data Processing | 253 |
| [no-global-plan-chain-of-thought](skills/no-global-plan-chain-of-thought/SKILL.md) | Reasoning & Chain-of-Thought, Efficiency & Optimization | 196 |
| [noir-privacy-preserving-generation-code](skills/noir-privacy-preserving-generation-code/SKILL.md) | Security & Safety, Code & Software Engineering, Prompt Engineering | 223 |
| [noisy-but-valid-robust](skills/noisy-but-valid-robust/SKILL.md) | Other | 242 |
| [note2chat-improving-multi-turn-clinical](skills/note2chat-improving-multi-turn-clinical/SKILL.md) | Reasoning & Chain-of-Thought, Agentic Systems, Data Processing +1 | 175 |
| [now-you-hear-me](skills/now-you-hear-me/SKILL.md) | Security & Safety, Evaluation & Benchmarking, Multimodal +1 | 311 |
| [nwa-mending-spatial-integrity-torn](skills/nwa-mending-spatial-integrity-torn/SKILL.md) | Multimodal, Efficiency & Optimization | 234 |
| [odysseyarena-benchmarking-long-horizon-active](skills/odysseyarena-benchmarking-long-horizon-active/SKILL.md) | Evaluation & Benchmarking, Agentic Systems | 176 |
| [omg-agent-robust-missing-modality](skills/omg-agent-robust-missing-modality/SKILL.md) | Agentic Systems | 280 |
| [omni-rrm-advancing-omni-reward](skills/omni-rrm-advancing-omni-reward/SKILL.md) | Evaluation & Benchmarking, Fine-tuning & Training, Multimodal +1 | 180 |
| [omni-safety-under-cross-modality-conflict](skills/omni-safety-under-cross-modality-conflict/SKILL.md) | Security & Safety, Multimodal, Data Processing | 217 |
| [omnicode-benchmark-evaluating-software](skills/omnicode-benchmark-evaluating-software/SKILL.md) | Code & Software Engineering, Evaluation & Benchmarking, Agentic Systems | 199 |
| [omnirag-agent-agentic-omnimodal-reasoning](skills/omnirag-agent-agentic-omnimodal-reasoning/SKILL.md) | RAG & Retrieval, Reasoning & Chain-of-Thought, Fine-tuning & Training +4 | 278 |
| [omnireview-large-scale-benchmark-llm-enhanced](skills/omnireview-large-scale-benchmark-llm-enhanced/SKILL.md) | Evaluation & Benchmarking, Data Processing | 204 |
| [on-effectiveness-llm-specific-fine-tuning](skills/on-effectiveness-llm-specific-fine-tuning/SKILL.md) | Evaluation & Benchmarking, Fine-tuning & Training | 171 |
| [on-impact-agentsmd-files](skills/on-impact-agentsmd-files/SKILL.md) | Code & Software Engineering, Agentic Systems, Efficiency & Optimization | 196 |
| [on-impact-code-comments](skills/on-impact-code-comments/SKILL.md) | Other | 219 |
| [on-uncertainty-model-based-multi-agent](skills/on-uncertainty-model-based-multi-agent/SKILL.md) | Multi-Agent Systems, Agentic Systems, Efficiency & Optimization | 184 |
| [on-use-generate-dataset](skills/on-use-generate-dataset/SKILL.md) | Evaluation & Benchmarking, Data Processing | 195 |
| [on-use-support-conduction](skills/on-use-support-conduction/SKILL.md) | RAG & Retrieval, Data Processing | 167 |
| [one-size-many-fits](skills/one-size-many-fits/SKILL.md) | RAG & Retrieval, Fine-tuning & Training, Multimodal +2 | 236 |
| [ontology-to-tools-compilation-executable-semantic-](skills/ontology-to-tools-compilation-executable-semantic/SKILL.md) | Code & Software Engineering, Knowledge Graphs, Agentic Systems +1 | 192 |
| [open-tutorai-open-source-platform](skills/open-tutorai-open-source-platform/SKILL.md) | Prompt Engineering, NLP & Text | 266 |
| [openguandan-large-scale-imperfect-information](skills/openguandan-large-scale-imperfect-information/SKILL.md) | Multi-Agent Systems, Evaluation & Benchmarking, Agentic Systems +1 | 366 |
| [opinf-llm-parametric-pde-solving](skills/opinf-llm-parametric-pde-solving/SKILL.md) | Evaluation & Benchmarking | 200 |
| [opportunities-aiml-rubin-lsst](skills/opportunities-aiml-rubin-lsst/SKILL.md) | Evaluation & Benchmarking, Data Processing | 249 |
| [optimal-turkish-subword-strategies](skills/optimal-turkish-subword-strategies/SKILL.md) | Reasoning & Chain-of-Thought, Evaluation & Benchmarking, Efficiency & Optimization | 253 |
| [optimizing-prompts-causal-approach](skills/optimizing-prompts-causal-approach/SKILL.md) | RAG & Retrieval, Prompt Engineering, Efficiency & Optimization +1 | 156 |
| [optimizing-small-sample-experience-learning-llm-ba](skills/optimizing-small-sample-experience-learning-llm-ba/SKILL.md) | Multi-Agent Systems, Fine-tuning & Training, Multimodal +3 | 195 |
| [orthogonal-hierarchical-decomposition-structure-aw](skills/orthogonal-hierarchical-decomposition-structure-aw/SKILL.md) | Data Processing, NLP & Text | 231 |
| [out-memory-barrier-highly](skills/out-memory-barrier-highly/SKILL.md) | Fine-tuning & Training, Memory & Context, Efficiency & Optimization | 200 |
| [outrunning-cutoffs-live-kernel](skills/outrunning-cutoffs-live-kernel/SKILL.md) | Code & Software Engineering, Evaluation & Benchmarking, Agentic Systems | 153 |
| [pabu-progress-aware-belief-update](skills/pabu-progress-aware-belief-update/SKILL.md) | Agentic Systems, Efficiency & Optimization | 210 |
| [pamas-self-adaptive-multi-agent-system](skills/pamas-self-adaptive-multi-agent-system/SKILL.md) | Multi-Agent Systems, Agentic Systems | 162 |
| [pand-prompt-aware-neighborhood-distillation](skills/pand-prompt-aware-neighborhood-distillation/SKILL.md) | Fine-tuning & Training, Multimodal, Prompt Engineering +1 | 189 |
| [paperbanana-automating-academic-illustration](skills/paperbanana-automating-academic-illustration/SKILL.md) | Multi-Agent Systems, RAG & Retrieval, Agentic Systems +1 | 197 |
| [papersearchqa-learning-search-reason](skills/papersearchqa-learning-search-reason/SKILL.md) | RAG & Retrieval, Reasoning & Chain-of-Thought, Agentic Systems +1 | 233 |
| [parameter-efficient-multi-task-fine-tuning-code-re](skills/parameter-efficient-multi-task-fine-tuning-code-re/SKILL.md) | Code & Software Engineering, Fine-tuning & Training, Efficiency & Optimization +1 | 223 |
| [parse-open-domain-reasoning-question](skills/parse-open-domain-reasoning-question/SKILL.md) | Reasoning & Chain-of-Thought, Evaluation & Benchmarking, Fine-tuning & Training +4 | 220 |
| [patch-to-poc-systematic-study-agentic](skills/patch-to-poc-systematic-study-agentic/SKILL.md) | Agentic Systems | 172 |
| [pathreasoner-r1-instilling-structured-reasoning](skills/pathreasoner-r1-instilling-structured-reasoning/SKILL.md) | Reasoning & Chain-of-Thought, Fine-tuning & Training, Multimodal +3 | 296 |
| [pathwise-planning-world-automated](skills/pathwise-planning-world-automated/SKILL.md) | Multi-Agent Systems, RAG & Retrieval, Evaluation & Benchmarking +2 | 165 |
| [pcbschemagen-constraint-guided-schematic-design](skills/pcbschemagen-constraint-guided-schematic-design/SKILL.md) | Code & Software Engineering, Data Processing | 209 |
| [pcl-reasoner-v15-advancing-math-reasoning-offline](skills/pcl-reasoner-v15-advancing-math-reasoning-offline/SKILL.md) | Reasoning & Chain-of-Thought, Fine-tuning & Training, Efficiency & Optimization +1 | 255 |
| [pearl-plan-exploration-adaptive](skills/pearl-plan-exploration-adaptive/SKILL.md) | Multi-Agent Systems, Agentic Systems, Data Processing | 172 |
| [pearl-prototype-enhanced-alignment-label-efficient](skills/pearl-prototype-enhanced-alignment-label-efficient/SKILL.md) | Efficiency & Optimization | 222 |
| [peerrank-autonomous-evaluation-web-grounded](skills/peerrank-autonomous-evaluation-web-grounded/SKILL.md) | Evaluation & Benchmarking, Agentic Systems, Data Processing | 215 |
| [perfguard-performance-aware-agent-visual](skills/perfguard-performance-aware-agent-visual/SKILL.md) | Multimodal, Agentic Systems | 227 |
| [persodpo-scalable-preference-optimization](skills/persodpo-scalable-preference-optimization/SKILL.md) | Evaluation & Benchmarking, Fine-tuning & Training, Efficiency & Optimization +1 | 173 |
| [persona-driven-data-synthesis-robust-multimodal](skills/persona-driven-data-synthesis-robust-multimodal/SKILL.md) | Reasoning & Chain-of-Thought, Fine-tuning & Training, Multimodal | 161 |
| [persona-jailbreaking](skills/persona-jailbreaking/SKILL.md) | Security & Safety, Evaluation & Benchmarking | 301 |
| [personality-as-relational-infrastructure](skills/personality-as-relational-infrastructure/SKILL.md) | Prompt Engineering, Efficiency & Optimization | 252 |
| [phenolip-integrating-phenotype-ontology](skills/phenolip-integrating-phenotype-ontology/SKILL.md) | RAG & Retrieval, Fine-tuning & Training, Multimodal +3 | 234 |
| [phostream-benchmarking-real-world-streaming](skills/phostream-benchmarking-real-world-streaming/SKILL.md) | Reasoning & Chain-of-Thought, Evaluation & Benchmarking, Multimodal +1 | 238 |
| [physical-prompt-injection-attacks](skills/physical-prompt-injection-attacks/SKILL.md) | Security & Safety, Prompt Engineering | 291 |
| [physprover-advancing-automatic-theorem](skills/physprover-advancing-automatic-theorem/SKILL.md) | Reasoning & Chain-of-Thought, Fine-tuning & Training, Data Processing | 218 |
| [planner-auditor-twin-agentic-discharge](skills/planner-auditor-twin-agentic-discharge/SKILL.md) | Agentic Systems, Data Processing | 189 |
| [polarmem-training-free-polarized-latent](skills/polarmem-training-free-polarized-latent/SKILL.md) | RAG & Retrieval, Reasoning & Chain-of-Thought, Fine-tuning & Training +4 | 183 |
| [pop-prefill-only-pruning-inference](skills/pop-prefill-only-pruning-inference/SKILL.md) | Efficiency & Optimization | 193 |
| [pope-learning-reason-hard](skills/pope-learning-reason-hard/SKILL.md) | Reasoning & Chain-of-Thought | 132 |
| [precise-reducing-bias-evaluations](skills/precise-reducing-bias-evaluations/SKILL.md) | RAG & Retrieval, Evaluation & Benchmarking, Data Processing | 212 |
| [precision-practice-knowledge-guided](skills/precision-practice-knowledge-guided/SKILL.md) | Other | 298 |
| [predicting-improving-test-time-scaling](skills/predicting-improving-test-time-scaling/SKILL.md) | RAG & Retrieval, Efficiency & Optimization | 209 |
| [predicting-intermittent-job-failure](skills/predicting-intermittent-job-failure/SKILL.md) | Prompt Engineering, Data Processing | 280 |
| [predictive-coding-information-bottleneck](skills/predictive-coding-information-bottleneck/SKILL.md) | Other | 238 |
| [prism-principled-framework-multi-agent](skills/prism-principled-framework-multi-agent/SKILL.md) | Multi-Agent Systems, Agentic Systems | 138 |
| [prism-xr-empowering-privacy-aware-xr](skills/prism-xr-empowering-privacy-aware-xr/SKILL.md) | Multi-Agent Systems, Multimodal, Data Processing | 301 |
| [privacy-collapse-benign-fine-tuning](skills/privacy-collapse-benign-fine-tuning/SKILL.md) | Security & Safety, Reasoning & Chain-of-Thought, Evaluation & Benchmarking +2 | 195 |
| [proact-agentic-lookahead-interactive](skills/proact-agentic-lookahead-interactive/SKILL.md) | Agentic Systems | 241 |
| [probing-knowledge-boundary-interactive](skills/probing-knowledge-boundary-interactive/SKILL.md) | Other | 181 |
| [profinfer-ebpf-based-fine-grained-inference](skills/profinfer-ebpf-based-fine-grained-inference/SKILL.md) | Memory & Context | 289 |
| [prograph-r1-progress-aware-reinforcement-learning](skills/prograph-r1-progress-aware-reinforcement-learning/SKILL.md) | RAG & Retrieval, Reasoning & Chain-of-Thought, Knowledge Graphs +2 | 167 |
| [prompt-augmentation-scales-up](skills/prompt-augmentation-scales-up/SKILL.md) | Prompt Engineering | 274 |
| [prompt-driven-development-claude](skills/prompt-driven-development-claude/SKILL.md) | Prompt Engineering | 188 |
| [prompt-injection-attacks-agentic](skills/prompt-injection-attacks-agentic/SKILL.md) | Security & Safety, Agentic Systems, Prompt Engineering | 207 |
| [promptrl-prompt-matters-rl](skills/promptrl-prompt-matters-rl/SKILL.md) | Fine-tuning & Training, Multimodal, Prompt Engineering +1 | 196 |
| [proopf-benchmarking-improving-professional-grade](skills/proopf-benchmarking-improving-professional-grade/SKILL.md) | Security & Safety, Evaluation & Benchmarking, Efficiency & Optimization +1 | 218 |
| [prorag-process-supervised-reinforcement-learning](skills/prorag-process-supervised-reinforcement-learning/SKILL.md) | RAG & Retrieval | 157 |
| [protean-compiler-agile-framework](skills/protean-compiler-agile-framework/SKILL.md) | RAG & Retrieval, Code & Software Engineering, Evaluation & Benchmarking +2 | 149 |
| [protoken-token-level-attribution-federated](skills/protoken-token-level-attribution-federated/SKILL.md) | Security & Safety, Code & Software Engineering, Fine-tuning & Training +2 | 164 |
| [proxywar-dynamic-assessment-of](skills/proxywar-dynamic-assessment-of/SKILL.md) | Code & Software Engineering, Evaluation & Benchmarking, Agentic Systems +1 | 197 |
| [pruning-minimal-reasoning-graphs](skills/pruning-minimal-reasoning-graphs/SKILL.md) | Reasoning & Chain-of-Thought, Efficiency & Optimization | 154 |
| [puda-private-user-dataset](skills/puda-private-user-dataset/SKILL.md) | Agentic Systems, Data Processing | 213 |
| [pull-requests-as-training](skills/pull-requests-as-training/SKILL.md) | RAG & Retrieval, Code & Software Engineering, Fine-tuning & Training +1 | 199 |
| [qrs-rule-synthesizing-neuro-symbolic-triad](skills/qrs-rule-synthesizing-neuro-symbolic-triad/SKILL.md) | Security & Safety, Reasoning & Chain-of-Thought, Agentic Systems | 229 |
| [quasar-universal-autonomous-system](skills/quasar-universal-autonomous-system/SKILL.md) | Multi-Agent Systems, RAG & Retrieval, Evaluation & Benchmarking +4 | 165 |
| [query-efficient-agentic-graph-extraction](skills/query-efficient-agentic-graph-extraction/SKILL.md) | Agentic Systems, Efficiency & Optimization, Data Processing | 239 |
| [r1-syntheticvl-synthetic-data-generative](skills/r1-syntheticvl-synthetic-data-generative/SKILL.md) | Security & Safety, Reasoning & Chain-of-Thought, Fine-tuning & Training +3 | 226 |
| [raca-representation-aware-coverage-criteria](skills/raca-representation-aware-coverage-criteria/SKILL.md) | RAG & Retrieval, Security & Safety, Evaluation & Benchmarking +1 | 242 |
| [ragturk-best-practices-retrieval](skills/ragturk-best-practices-retrieval/SKILL.md) | RAG & Retrieval, Reasoning & Chain-of-Thought, Efficiency & Optimization +1 | 208 |
| [raicl-retrieval-augmented-in-context-learning](skills/raicl-retrieval-augmented-in-context-learning/SKILL.md) | RAG & Retrieval, Multimodal, Prompt Engineering +1 | 228 |
| [ral-bench-benchmarking-application-level-functiona](skills/ral-bench-benchmarking-application-level-functiona/SKILL.md) | Security & Safety, Code & Software Engineering, Evaluation & Benchmarking +2 | 179 |
| [rank-and-reason-multi-agent-collaboration-accelera](skills/rank-and-reason-multi-agent-collaboration-accelera/SKILL.md) | Multi-Agent Systems, Reasoning & Chain-of-Thought, Agentic Systems | 245 |
| [rapid-real-time-deterministic-trajectory](skills/rapid-real-time-deterministic-trajectory/SKILL.md) | Security & Safety, Fine-tuning & Training, Multimodal +4 | 185 |
| [rapo-risk-aware-preference-optimization](skills/rapo-risk-aware-preference-optimization/SKILL.md) | Security & Safety, Reasoning & Chain-of-Thought, Fine-tuning & Training +3 | 203 |
| [rate-reviewer-profiling-annotation-free](skills/rate-reviewer-profiling-annotation-free/SKILL.md) | Data Processing | 224 |
| [rc-grpo-reward-conditioned-group-relative](skills/rc-grpo-reward-conditioned-group-relative/SKILL.md) | Fine-tuning & Training, Agentic Systems, Prompt Engineering +2 | 228 |
| [read-as-human-compressing](skills/read-as-human-compressing/SKILL.md) | Memory & Context, Efficiency & Optimization, NLP & Text | 242 |
| [realhd-high-quality-dataset-robust](skills/realhd-high-quality-dataset-robust/SKILL.md) | Multimodal, Data Processing | 230 |
| [realistic-synthetic-household-data](skills/realistic-synthetic-household-data/SKILL.md) | Fine-tuning & Training, Data Processing | 179 |
| [realsec-bench-benchmark-evaluating-secure](skills/realsec-bench-benchmark-evaluating-secure/SKILL.md) | Evaluation & Benchmarking | 195 |
| [reasoning-augmented-representations-multimodal-ret](skills/reasoning-augmented-representations-multimodal-ret/SKILL.md) | RAG & Retrieval, Reasoning & Chain-of-Thought, Multimodal +2 | 223 |
| [reasoning-beyond-literal-cross-style](skills/reasoning-beyond-literal-cross-style/SKILL.md) | Reasoning & Chain-of-Thought, Multimodal, Explainability +1 | 176 |
| [reasoning-tool-use-compete-agentic](skills/reasoning-tool-use-compete-agentic/SKILL.md) | Reasoning & Chain-of-Thought, Fine-tuning & Training, Agentic Systems +2 | 204 |
| [reasoning-while-asking-transforming](skills/reasoning-while-asking-transforming/SKILL.md) | Reasoning & Chain-of-Thought | 190 |
| [rebel-hidden-knowledge-recovery](skills/rebel-hidden-knowledge-recovery/SKILL.md) | RAG & Retrieval, Evaluation & Benchmarking, Prompt Engineering +1 | 28 |
| [recgoat-graph-optimal-adaptive](skills/recgoat-graph-optimal-adaptive/SKILL.md) | Memory & Context, Multimodal | 182 |
| [reconstructing-training-data-adapter-based](skills/reconstructing-training-data-adapter-based/SKILL.md) | Fine-tuning & Training | 177 |
| [redsage-cybersecurity-generalist](skills/redsage-cybersecurity-generalist/SKILL.md) | Security & Safety | 234 |
| [reducing-costs-proof-synthesis](skills/reducing-costs-proof-synthesis/SKILL.md) | Code & Software Engineering, Reasoning & Chain-of-Thought | 247 |
| [redvisor-reasoning-aware-prompt-injection](skills/redvisor-reasoning-aware-prompt-injection/SKILL.md) | RAG & Retrieval, Security & Safety, Reasoning & Chain-of-Thought +4 | 223 |
| [refer-agent-collaborative-multi-agent-system](skills/refer-agent-collaborative-multi-agent-system/SKILL.md) | Multi-Agent Systems, Reasoning & Chain-of-Thought, Agentic Systems +1 | 179 |
| [refining-decision-boundaries-anomaly](skills/refining-decision-boundaries-anomaly/SKILL.md) | Other | 178 |
| [reflect-transparent-principle-guided-reasoning](skills/reflect-transparent-principle-guided-reasoning/SKILL.md) | Reasoning & Chain-of-Thought, Explainability | 185 |
| [refuge-feature-generation-prediction](skills/refuge-feature-generation-prediction/SKILL.md) | Multi-Agent Systems, Agentic Systems, Data Processing | 168 |
| [regular-variational-latent-reasoning](skills/regular-variational-latent-reasoning/SKILL.md) | Reasoning & Chain-of-Thought, Multimodal, Efficiency & Optimization | 236 |
| [reinforced-attention-learning](skills/reinforced-attention-learning/SKILL.md) | Fine-tuning & Training, Memory & Context, Multimodal +2 | 216 |
| [reinforcement-learning-backtracking-feedback](skills/reinforcement-learning-backtracking-feedback/SKILL.md) | Security & Safety, Fine-tuning & Training | 289 |
| [reinforcement-learning-self-distillation](skills/reinforcement-learning-self-distillation/SKILL.md) | Fine-tuning & Training, Efficiency & Optimization | 252 |
| [reliable-responsible-foundation-comprehensive](skills/reliable-responsible-foundation-comprehensive/SKILL.md) | Other | 360 |
| [remedit-diffusion-editing-riemannian](skills/remedit-diffusion-editing-riemannian/SKILL.md) | Memory & Context, Multimodal, Prompt Engineering +2 | 244 |
| [render-of-thought-rendering-textual-chain-of-thoug](skills/render-of-thought-rendering-textual-chain-of-thoug/SKILL.md) | Other | 218 |
| [report-nsf-workshop-ai](skills/report-nsf-workshop-ai/SKILL.md) | RAG & Retrieval, Code & Software Engineering, Reasoning & Chain-of-Thought +1 | 276 |
| [reprompt-prompt-generation-intelligent](skills/reprompt-prompt-generation-intelligent/SKILL.md) | Agentic Systems, Prompt Engineering, Efficiency & Optimization | 191 |
| [resagent-entropy-based-prior-point](skills/resagent-entropy-based-prior-point/SKILL.md) | Reasoning & Chain-of-Thought, Multimodal, Agentic Systems +2 | 266 |
| [research-multi-stage-machine-learning](skills/research-multi-stage-machine-learning/SKILL.md) | RAG & Retrieval, Data Processing | 279 |
| [residual-context-diffusion](skills/residual-context-diffusion/SKILL.md) | Efficiency & Optimization | 193 |
| [rethinker-scientific-reasoning-rethinking](skills/rethinker-scientific-reasoning-rethinking/SKILL.md) | Reasoning & Chain-of-Thought | 239 |
| [rethinking-generative-recommender-tokenizer](skills/rethinking-generative-recommender-tokenizer/SKILL.md) | Efficiency & Optimization | 154 |
| [rethinking-genomic-modeling-optical](skills/rethinking-genomic-modeling-optical/SKILL.md) | Multimodal, Efficiency & Optimization, Data Processing | 248 |
| [rethinking-irregular-time-series](skills/rethinking-irregular-time-series/SKILL.md) | Efficiency & Optimization, Data Processing, Domain-Specific | 186 |
| [rethinking-llm-as-a-judge-representation-as-a-judg](skills/rethinking-llm-as-a-judge-representation-as-a-judg/SKILL.md) | Reasoning & Chain-of-Thought, Evaluation & Benchmarking, Fine-tuning & Training +2 | 160 |
| [rethinking-perplexity-revealing-impact](skills/rethinking-perplexity-revealing-impact/SKILL.md) | Other | 303 |
| [rethinking-reranker-boundary-aware-evidence](skills/rethinking-reranker-boundary-aware-evidence/SKILL.md) | RAG & Retrieval | 159 |
| [rethinking-role-entropy-optimizing](skills/rethinking-role-entropy-optimizing/SKILL.md) | Agentic Systems, Efficiency & Optimization | 175 |
| [rethinking-scientific-modeling-physically](skills/rethinking-scientific-modeling-physically/SKILL.md) | Code & Software Engineering, Evaluation & Benchmarking | 209 |
| [rethinking-trust-region-reinforcement](skills/rethinking-trust-region-reinforcement/SKILL.md) | Fine-tuning & Training, Efficiency & Optimization | 204 |
| [rethinking-value-agent-generated-tests](skills/rethinking-value-agent-generated-tests/SKILL.md) | Code & Software Engineering, Agentic Systems, Efficiency & Optimization | 163 |
| [reverse-engineering-editing](skills/reverse-engineering-editing/SKILL.md) | Security & Safety | 182 |
| [revisiting-adaptive-rounding-vectorized](skills/revisiting-adaptive-rounding-vectorized/SKILL.md) | Fine-tuning & Training, Efficiency & Optimization, Data Processing | 175 |
| [revisiting-role-natural-code](skills/revisiting-role-natural-code/SKILL.md) | NLP & Text | 215 |
| [revisiting-salient-object-detection](skills/revisiting-salient-object-detection/SKILL.md) | Multimodal, Agentic Systems, Data Processing | 257 |
| [reward-designs-general-reasoning](skills/reward-designs-general-reasoning/SKILL.md) | Reasoning & Chain-of-Thought, Fine-tuning & Training | 228 |
| [reward-free-alignment-conflicting-objectives](skills/reward-free-alignment-conflicting-objectives/SKILL.md) | Security & Safety, Fine-tuning & Training | 239 |
| [reward-inherit-value-biases](skills/reward-inherit-value-biases/SKILL.md) | Fine-tuning & Training | 178 |
| [rewards-as-labels-revisiting](skills/rewards-as-labels-revisiting/SKILL.md) | Reasoning & Chain-of-Thought, Fine-tuning & Training, Efficiency & Optimization | 218 |
| [risk-awareness-injection-calibrating](skills/risk-awareness-injection-calibrating/SKILL.md) | Security & Safety, Fine-tuning & Training, Multimodal +1 | 244 |
| [robust-tool-use-fission-grpo](skills/robust-tool-use-fission-grpo/SKILL.md) | Fine-tuning & Training | 320 |
| [robustexplain-evaluating-robustness-llm-based](skills/robustexplain-evaluating-robustness-llm-based/SKILL.md) | Evaluation & Benchmarking, Agentic Systems, Data Processing +1 | 211 |
| [roma-recursive-open-meta-agent](skills/roma-recursive-open-meta-agent/SKILL.md) | Multi-Agent Systems, Agentic Systems, Efficiency & Optimization +1 | 185 |
| [rpo-rag-aligning-small-relation-aware](skills/rpo-rag-aligning-small-relation-aware/SKILL.md) | RAG & Retrieval, Reasoning & Chain-of-Thought, Knowledge Graphs +3 | 259 |
| [rubberduckbench-benchmark-ai-coding](skills/rubberduckbench-benchmark-ai-coding/SKILL.md) | Reasoning & Chain-of-Thought, Evaluation & Benchmarking | 182 |
| [ruleflow-generating-reusable-program](skills/ruleflow-generating-reusable-program/SKILL.md) | Code & Software Engineering, Efficiency & Optimization, Data Processing | 160 |
| [rulesmith-multi-agent-automated-game](skills/rulesmith-multi-agent-automated-game/SKILL.md) | Multi-Agent Systems, Agentic Systems, Efficiency & Optimization +1 | 193 |
| [rvb-automating-ai-system](skills/rvb-automating-ai-system/SKILL.md) | Security & Safety, Efficiency & Optimization | 222 |
| [rvcbench-benchmarking-robustness-voice](skills/rvcbench-benchmarking-robustness-voice/SKILL.md) | Security & Safety, Evaluation & Benchmarking, Multimodal +2 | 164 |
| [s3-cot-self-sampled-succinct-reasoning](skills/s3-cot-self-sampled-succinct-reasoning/SKILL.md) | Reasoning & Chain-of-Thought, Efficiency & Optimization | 201 |
| [safepred-predictive-guardrail-computer-using](skills/safepred-predictive-guardrail-computer-using/SKILL.md) | Security & Safety, Agentic Systems, Data Processing | 179 |
| [sar-rag-atr-visual-question](skills/sar-rag-atr-visual-question/SKILL.md) | RAG & Retrieval, Multimodal | 378 |
| [scalable-generative-game-engine](skills/scalable-generative-game-engine/SKILL.md) | Memory & Context, Efficiency & Optimization, Data Processing | 162 |
| [scaled-surrogate-gradient-codec-aware-learning](skills/scaled-surrogate-gradient-codec-aware-learning/SKILL.md) | Fine-tuning & Training, Multimodal, Efficiency & Optimization +1 | 215 |
| [scaling-medical-reasoning-verification](skills/scaling-medical-reasoning-verification/SKILL.md) | Reasoning & Chain-of-Thought, Domain-Specific | 191 |
| [scaling-up-privacy-preserving-ml](skills/scaling-up-privacy-preserving-ml/SKILL.md) | Other | 194 |
| [scidatacopilot-agentic-data-preparation](skills/scidatacopilot-agentic-data-preparation/SKILL.md) | RAG & Retrieval, Agentic Systems, Data Processing | 240 |
| [scratcheval-multimodal-evaluation-framework](skills/scratcheval-multimodal-evaluation-framework/SKILL.md) | Code & Software Engineering, Reasoning & Chain-of-Thought, Evaluation & Benchmarking +1 | 156 |
| [sdr-cir-semantic-debias-retrieval](skills/sdr-cir-semantic-debias-retrieval/SKILL.md) | RAG & Retrieval, Fine-tuning & Training, Multimodal +1 | 171 |
| [se-bench-benchmarking-self-evolution-knowledge](skills/se-bench-benchmarking-self-evolution-knowledge/SKILL.md) | Evaluation & Benchmarking, Fine-tuning & Training, Data Processing | 163 |
| [seccodeprm-process-reward-code](skills/seccodeprm-process-reward-code/SKILL.md) | Security & Safety, Code & Software Engineering, Evaluation & Benchmarking +1 | 201 |
| [secure-code-generation-via](skills/secure-code-generation-via/SKILL.md) | Other | 254 |
| [sed-sft-selectively-encouraging-diversity](skills/sed-sft-selectively-encouraging-diversity/SKILL.md) | RAG & Retrieval, Fine-tuning & Training | 229 |
| [selective-steering-norm-preserving-control](skills/selective-steering-norm-preserving-control/SKILL.md) | Security & Safety, Data Processing | 174 |
| [self-evolving-recommendation-system-end-to-end](skills/self-evolving-recommendation-system-end-to-end/SKILL.md) | RAG & Retrieval, Evaluation & Benchmarking, Agentic Systems +2 | 168 |
| [self-hinting-enhance-reinforcement-learning](skills/self-hinting-enhance-reinforcement-learning/SKILL.md) | Reasoning & Chain-of-Thought, Fine-tuning & Training, Prompt Engineering +1 | 174 |
| [self-improving-pretraining-post-trained-pretrain](skills/self-improving-pretraining-post-trained-pretrain/SKILL.md) | Security & Safety, Evaluation & Benchmarking, Fine-tuning & Training +1 | 256 |
| [self-improving-world-modelling-latent](skills/self-improving-world-modelling-latent/SKILL.md) | Other | 181 |
| [sema-simple-yet-learning](skills/sema-simple-yet-learning/SKILL.md) | Other | 256 |
| [semantic-aware-advanced-persistent-threat](skills/semantic-aware-advanced-persistent-threat/SKILL.md) | Other | 194 |
| [semanticalli-caching-reasoning-not](skills/semanticalli-caching-reasoning-not/SKILL.md) | Reasoning & Chain-of-Thought, Agentic Systems, Data Processing | 202 |
| [semantically-aware-uav-landing](skills/semantically-aware-uav-landing/SKILL.md) | Other | 231 |
| [sera-soft-verified-repository-agents](skills/sera-soft-verified-repository-agents/SKILL.md) | Code & Software Engineering, Agentic Systems | 220 |
| [sere-similarity-based-expert-re-routing](skills/sere-similarity-based-expert-re-routing/SKILL.md) | Efficiency & Optimization | 223 |
| [seta-statistical-fault-attribution](skills/seta-statistical-fault-attribution/SKILL.md) | Data Processing, Explainability | 247 |
| [shardmemo-masked-moe-routing](skills/shardmemo-masked-moe-routing/SKILL.md) | Multi-Agent Systems, RAG & Retrieval, Memory & Context +1 | 176 |
| [shieldedcode-learning-robust-representations](skills/shieldedcode-learning-robust-representations/SKILL.md) | Other | 193 |
| [shine-scalable-in-context-hypernetwork](skills/shine-scalable-in-context-hypernetwork/SKILL.md) | Fine-tuning & Training, Memory & Context, Efficiency & Optimization | 225 |
| [shopsimulator-evaluating-exploring-rl-driven](skills/shopsimulator-evaluating-exploring-rl-driven/SKILL.md) | RAG & Retrieval, Evaluation & Benchmarking, Fine-tuning & Training +1 | 285 |
| [shotfinder-imagination-driven-open-domain-video](skills/shotfinder-imagination-driven-open-domain-video/SKILL.md) | Multimodal | 189 |
| [sicl-at-another-way-adapt](skills/sicl-at-another-way-adapt/SKILL.md) | Fine-tuning & Training, Multimodal, Prompt Engineering | 168 |
| [sifting-noise-comparative-study](skills/sifting-noise-comparative-study/SKILL.md) | Security & Safety, Reasoning & Chain-of-Thought, Agentic Systems | 154 |
| [skillrl-evolving-agents-recursive](skills/skillrl-evolving-agents-recursive/SKILL.md) | Fine-tuning & Training, Agentic Systems, Prompt Engineering +2 | 184 |
| [skyreels-v3-technique-report](skills/skyreels-v3-technique-report/SKILL.md) | Other | 279 |
| [small-beautiful-practical-log](skills/small-beautiful-practical-log/SKILL.md) | Efficiency & Optimization, Data Processing | 191 |
| [smartoracle-agentic-approach](skills/smartoracle-agentic-approach/SKILL.md) | Code & Software Engineering, Agentic Systems, Data Processing | 177 |
| [snapmla-efficient-longcontext-mla](skills/snapmla-efficient-longcontext-mla/SKILL.md) | RAG & Retrieval, Code & Software Engineering, Memory & Context +3 | 88 |
| [snapmla-long-context-mla-decoding](skills/snapmla-long-context-mla-decoding/SKILL.md) | Memory & Context, Efficiency & Optimization, Data Processing | 186 |
| [social-catalysts-not-moral](skills/social-catalysts-not-moral/SKILL.md) | Other | 180 |
| [socialveil-probing-social-intelligence](skills/socialveil-probing-social-intelligence/SKILL.md) | Multi-Agent Systems, Evaluation & Benchmarking, Agentic Systems | 203 |
| [socratic-geo-synthetic-data-generation](skills/socratic-geo-synthetic-data-generation/SKILL.md) | Multi-Agent Systems, Reasoning & Chain-of-Thought, Evaluation & Benchmarking +3 | 226 |
| [sogk-one-token-explicit-graph](skills/sogk-one-token-explicit-graph/SKILL.md) | Reasoning & Chain-of-Thought, Prompt Engineering | 128 |
| [sogptspotter-detecting-chatgpt-generated-answers](skills/sogptspotter-detecting-chatgpt-generated-answers/SKILL.md) | Other | 200 |
| [solagent-specialized-multi-agent-framework](skills/solagent-specialized-multi-agent-framework/SKILL.md) | Multi-Agent Systems, Security & Safety, Code & Software Engineering +2 | 193 |
| [sonic-o1-real-world-benchmark-evaluating](skills/sonic-o1-real-world-benchmark-evaluating/SKILL.md) | Reasoning & Chain-of-Thought, Evaluation & Benchmarking, Multimodal +1 | 238 |
| [soundbreak-systematic-study-audio-only](skills/soundbreak-systematic-study-audio-only/SKILL.md) | Multimodal | 192 |
| [sparc-rag-adaptive-sequential-parallel-scaling](skills/sparc-rag-adaptive-sequential-parallel-scaling/SKILL.md) | Multi-Agent Systems, RAG & Retrieval, Agentic Systems +2 | 248 |
| [sparc-separating-perception-reasoning](skills/sparc-separating-perception-reasoning/SKILL.md) | Reasoning & Chain-of-Thought | 170 |
| [sparse-sparse-safety-unsafe](skills/sparse-sparse-safety-unsafe/SKILL.md) | Security & Safety, Evaluation & Benchmarking, Efficiency & Optimization | 184 |
| [sparseeval-evaluation-sparse-optimization](skills/sparseeval-evaluation-sparse-optimization/SKILL.md) | Evaluation & Benchmarking, Efficiency & Optimization, Data Processing | 221 |
| [spatialab-vision-language-perform-spatial](skills/spatialab-vision-language-perform-spatial/SKILL.md) | Multimodal | 242 |
| [spava-accelerating-long-video-understanding](skills/spava-accelerating-long-video-understanding/SKILL.md) | Memory & Context, Multimodal, Efficiency & Optimization +1 | 200 |
| [spd-faith-bench-diagnosing-improving](skills/spd-faith-bench-diagnosing-improving/SKILL.md) | Reasoning & Chain-of-Thought, Multimodal, Data Processing | 237 |
| [spectral-guardrails-agents-wild](skills/spectral-guardrails-agents-wild/SKILL.md) | Security & Safety, Reasoning & Chain-of-Thought, Fine-tuning & Training +2 | 250 |
| [spell-synthesis-programmatic-edits](skills/spell-synthesis-programmatic-edits/SKILL.md) | Fine-tuning & Training, Efficiency & Optimization | 209 |
| [spider-sense-intrinsic-risk-sensing](skills/spider-sense-intrinsic-risk-sensing/SKILL.md) | Security & Safety, Agentic Systems, Prompt Engineering +1 | 212 |
| [spotagent-grounding-visual-geo-localization](skills/spotagent-grounding-visual-geo-localization/SKILL.md) | Reasoning & Chain-of-Thought, Multimodal, Agentic Systems +1 | 248 |
| [sql-trail-multi-turn-reinforcement-learning](skills/sql-trail-multi-turn-reinforcement-learning/SKILL.md) | Code & Software Engineering, Reasoning & Chain-of-Thought | 186 |
| [sqlagent-learning-explore-before](skills/sqlagent-learning-explore-before/SKILL.md) | RAG & Retrieval, Agentic Systems | 201 |
| [st-raptor-agentic-system-semi-structured](skills/st-raptor-agentic-system-semi-structured/SKILL.md) | Reasoning & Chain-of-Thought, Agentic Systems, Data Processing +1 | 222 |
| [stalled-biased-confused-uncovering-reasoning](skills/stalled-biased-confused-uncovering-reasoning/SKILL.md) | Reasoning & Chain-of-Thought | 168 |
| [standardizing-longitudinal-radiology-report](skills/standardizing-longitudinal-radiology-report/SKILL.md) | Evaluation & Benchmarking, Data Processing, Domain-Specific | 230 |
| [star-similarity-guided-teacher-assisted-refinement](skills/star-similarity-guided-teacher-assisted-refinement/SKILL.md) | Fine-tuning & Training, Agentic Systems, Efficiency & Optimization | 281 |
| [state-art-llm-enabled-interaction](skills/state-art-llm-enabled-interaction/SKILL.md) | Multimodal, Data Processing, Explainability | 258 |
| [state-transition-framework-reasoning](skills/state-transition-framework-reasoning/SKILL.md) | Reasoning & Chain-of-Thought | 215 |
| [stateless-yet-not-forgetful](skills/stateless-yet-not-forgetful/SKILL.md) | Security & Safety, Memory & Context, Agentic Systems +1 | 245 |
| [status-hierarchies](skills/status-hierarchies/SKILL.md) | Multi-Agent Systems, Agentic Systems, Data Processing | 229 |
| [stealthrl-reinforcement-learning-paraphrase](skills/stealthrl-reinforcement-learning-paraphrase/SKILL.md) | Other | 345 |
| [steer2adapt-dynamically-composing-steering](skills/steer2adapt-dynamically-composing-steering/SKILL.md) | Security & Safety, Reasoning & Chain-of-Thought, Fine-tuning & Training +1 | 209 |
| [steereval-framework-evaluating-steerability](skills/steereval-framework-evaluating-steerability/SKILL.md) | Evaluation & Benchmarking, Data Processing | 191 |
| [steering-externalities-benign-activation](skills/steering-externalities-benign-activation/SKILL.md) | Security & Safety | 247 |
| [steering-safely-or-off](skills/steering-safely-or-off/SKILL.md) | Other | 207 |
| [step-35-flash-open](skills/step-35-flash-open/SKILL.md) | Fine-tuning & Training, Memory & Context, Agentic Systems +2 | 226 |
| [stepshield-not-whether-intervene](skills/stepshield-not-whether-intervene/SKILL.md) | Security & Safety, Agentic Systems | 282 |
| [steuerllm-local-specialized-german](skills/steuerllm-local-specialized-german/SKILL.md) | RAG & Retrieval, Reasoning & Chain-of-Thought, Evaluation & Benchmarking +3 | 203 |
| [stop-testing-attacks-start](skills/stop-testing-attacks-start/SKILL.md) | Security & Safety | 222 |
| [streaming-dllm-accelerating-diffusion-suffix](skills/streaming-dllm-accelerating-diffusion-suffix/SKILL.md) | Efficiency & Optimization | 238 |
| [strong-reasoning-isnt-enough](skills/strong-reasoning-isnt-enough/SKILL.md) | RAG & Retrieval, Reasoning & Chain-of-Thought, Evaluation & Benchmarking +1 | 251 |
| [structured-context-engineering-file-native](skills/structured-context-engineering-file-native/SKILL.md) | Other | 302 |
| [subliminal-effects-data-general](skills/subliminal-effects-data-general/SKILL.md) | Other | 214 |
| [supchain-bench-benchmarking-real-world-supply](skills/supchain-bench-benchmarking-real-world-supply/SKILL.md) | Multi-Agent Systems, Evaluation & Benchmarking, Agentic Systems | 198 |
| [supporting-software-engineering-tasks](skills/supporting-software-engineering-tasks/SKILL.md) | RAG & Retrieval, Code & Software Engineering, Agentic Systems +1 | 169 |
| [svrepair-structured-visual-reasoning](skills/svrepair-structured-visual-reasoning/SKILL.md) | Code & Software Engineering, Reasoning & Chain-of-Thought, Multimodal | 193 |
| [swe-agi-benchmarking-specification-driven-software](skills/swe-agi-benchmarking-specification-driven-software/SKILL.md) | Code & Software Engineering, Evaluation & Benchmarking | 189 |
| [swe-bench-mobile-agents-develop](skills/swe-bench-mobile-agents-develop/SKILL.md) | Code & Software Engineering, Agentic Systems | 247 |
| [swe-context-bench-benchmark](skills/swe-context-bench-benchmark/SKILL.md) | RAG & Retrieval, Code & Software Engineering, Evaluation & Benchmarking +1 | 180 |
| [swe-manager-selecting-synthesizing-golden](skills/swe-manager-selecting-synthesizing-golden/SKILL.md) | Code & Software Engineering | 254 |
| [swe-master-unleashing-potential-software](skills/swe-master-unleashing-potential-software/SKILL.md) | Code & Software Engineering | 195 |
| [swe-pruner-self-adaptive-context-pruning](skills/swe-pruner-self-adaptive-context-pruning/SKILL.md) | Code & Software Engineering, Efficiency & Optimization | 186 |
| [swe-refactor-repository-level-benchmark-real-world](skills/swe-refactor-repository-level-benchmark-real-world/SKILL.md) | Code & Software Engineering, Evaluation & Benchmarking | 151 |
| [swe-replay-test-time-scaling-software](skills/swe-replay-test-time-scaling-software/SKILL.md) | Code & Software Engineering, Agentic Systems, Efficiency & Optimization | 158 |
| [swe-spot-building-small-repo-experts](skills/swe-spot-building-small-repo-experts/SKILL.md) | Code & Software Engineering | 183 |
| [swe-world-building-software-engineering](skills/swe-world-building-software-engineering/SKILL.md) | Code & Software Engineering | 196 |
| [sycoeval-em-sycophancy-evaluation-simulated](skills/sycoeval-em-sycophancy-evaluation-simulated/SKILL.md) | Multi-Agent Systems, Security & Safety, Evaluation & Benchmarking +1 | 241 |
| [symphony-synergistic-multi-agent-planning](skills/symphony-synergistic-multi-agent-planning/SKILL.md) | Multi-Agent Systems, Agentic Systems | 201 |
| [syncabel-synthetic-contextualized-augmentation](skills/syncabel-synthetic-contextualized-augmentation/SKILL.md) | Fine-tuning & Training, Data Processing, Domain-Specific +1 | 193 |
| [synthagent-multi-agent-framework-realistic](skills/synthagent-multi-agent-framework-realistic/SKILL.md) | Multi-Agent Systems, Reasoning & Chain-of-Thought, Agentic Systems +2 | 298 |
| [synthesizing-file-level-data-unit](skills/synthesizing-file-level-data-unit/SKILL.md) | RAG & Retrieval, Code & Software Engineering, Reasoning & Chain-of-Thought | 222 |
| [system-12-synergy-dynamic](skills/system-12-synergy-dynamic/SKILL.md) | Other | 216 |
| [system-name-address-parsing](skills/system-name-address-parsing/SKILL.md) | Prompt Engineering, Data Processing | 207 |
| [t-llm-teaching-forecast-time](skills/t-llm-teaching-forecast-time/SKILL.md) | Fine-tuning & Training, Efficiency & Optimization, Data Processing | 155 |
| [t2vtree-user-centered-visual-analytics](skills/t2vtree-user-centered-visual-analytics/SKILL.md) | Multimodal | 268 |
| [table-as-search-formulate-long-horizon-agentic](skills/table-as-search-formulate-long-horizon-agentic/SKILL.md) | Multi-Agent Systems, RAG & Retrieval, Agentic Systems +1 | 203 |
| [tam-eval-evaluating-llms-for](skills/tam-eval-evaluating-llms-for/SKILL.md) | Evaluation & Benchmarking | 225 |
| [taming-scylla-understanding-multi-headed](skills/taming-scylla-understanding-multi-headed/SKILL.md) | Other | 165 |
| [tamperbench-systematically-stress-testing-safety](skills/tamperbench-systematically-stress-testing-safety/SKILL.md) | Security & Safety, Evaluation & Benchmarking, Fine-tuning & Training +1 | 250 |
| [tangrampuzzle-evaluating-multimodal-compositional](skills/tangrampuzzle-evaluating-multimodal-compositional/SKILL.md) | Reasoning & Chain-of-Thought, Evaluation & Benchmarking, Multimodal | 233 |
| [task-oriented-robot-human-handovers-legged](skills/task-oriented-robot-human-handovers-legged/SKILL.md) | Reasoning & Chain-of-Thought, Agentic Systems, Prompt Engineering +2 | 261 |
| [teaching-evaluating-reason-about](skills/teaching-evaluating-reason-about/SKILL.md) | Reasoning & Chain-of-Thought, Evaluation & Benchmarking, Fine-tuning & Training +2 | 202 |
| [team-temporal-spatial-consistency-guided](skills/team-temporal-spatial-consistency-guided/SKILL.md) | Efficiency & Optimization | 193 |
| [temp-r1-unified-autonomous-agent](skills/temp-r1-unified-autonomous-agent/SKILL.md) | RAG & Retrieval, Reasoning & Chain-of-Thought, Knowledge Graphs +3 | 200 |
| [termigen-high-fidelity-environment-robust](skills/termigen-high-fidelity-environment-robust/SKILL.md) | Multi-Agent Systems, Fine-tuning & Training, Agentic Systems | 181 |
| [ternarylm-memory-efficient-modeling-native](skills/ternarylm-memory-efficient-modeling-native/SKILL.md) | Fine-tuning & Training, Memory & Context, Efficiency & Optimization | 230 |
| [test-vs-mutant-adversarial](skills/test-vs-mutant-adversarial/SKILL.md) | Security & Safety | 221 |
| [testexplora-benchmarking-proactive-bug](skills/testexplora-benchmarking-proactive-bug/SKILL.md) | Code & Software Engineering, Evaluation & Benchmarking | 218 |
| [text-summarization-global-structure](skills/text-summarization-global-structure/SKILL.md) | Reasoning & Chain-of-Thought, Memory & Context, Efficiency & Optimization +2 | 165 |
| [textual-equilibrium-propagation-deep](skills/textual-equilibrium-propagation-deep/SKILL.md) | Other | 147 |
| [the-clef-2026-finmmeval-lab](skills/the-clef-2026-finmmeval-lab/SKILL.md) | Reasoning & Chain-of-Thought, Evaluation & Benchmarking, Multimodal +4 | 246 |
| [the-compliance-paradox-semantic-instruction](skills/the-compliance-paradox-semantic-instruction/SKILL.md) | Security & Safety, Evaluation & Benchmarking, Prompt Engineering | 229 |
| [the-effectiveness-style-vectors](skills/the-effectiveness-style-vectors/SKILL.md) | Data Processing | 276 |
| [the-landscape-prompt-injection](skills/the-landscape-prompt-injection/SKILL.md) | Security & Safety, Evaluation & Benchmarking, Agentic Systems +2 | 244 |
| [the-necessity-unified-framework](skills/the-necessity-unified-framework/SKILL.md) | Evaluation & Benchmarking, Agentic Systems, Prompt Engineering | 185 |
| [the-semantic-trap-fine-tuned](skills/the-semantic-trap-fine-tuned/SKILL.md) | Fine-tuning & Training | 179 |
| [the-shadow-self-intrinsic](skills/the-shadow-self-intrinsic/SKILL.md) | Security & Safety, Evaluation & Benchmarking, Agentic Systems +1 | 234 |
| [the-side-effects-being](skills/the-side-effects-being/SKILL.md) | Other | 205 |
| [think-augmented-function-calling-improving](skills/think-augmented-function-calling-improving/SKILL.md) | Other | 262 |
| [thinking-broad-acting-fast](skills/thinking-broad-acting-fast/SKILL.md) | RAG & Retrieval, Reasoning & Chain-of-Thought, Fine-tuning & Training +1 | 210 |
| [thinking-frames-visual-context](skills/thinking-frames-visual-context/SKILL.md) | Reasoning & Chain-of-Thought, Multimodal, Agentic Systems | 238 |
| [thinking-makes-agents-introverted](skills/thinking-makes-agents-introverted/SKILL.md) | Agentic Systems | 244 |
| [thinktank-me-multi-expert-framework-middle](skills/thinktank-me-multi-expert-framework-middle/SKILL.md) | Multi-Agent Systems, Agentic Systems | 207 |
| [timbre-aware-llm-based-direct-speech-to-speech](skills/timbre-aware-llm-based-direct-speech-to-speech/SKILL.md) | Multimodal, Data Processing, NLP & Text | 210 |
| [timeblind-spatio-temporal-compositionality-benchma](skills/timeblind-spatio-temporal-compositionality-benchma/SKILL.md) | Reasoning & Chain-of-Thought, Evaluation & Benchmarking, Multimodal | 233 |
| [timely-machine-awareness-time](skills/timely-machine-awareness-time/SKILL.md) | Reasoning & Chain-of-Thought, Agentic Systems, Efficiency & Optimization | 171 |
| [timemachine-bench-benchmark-evaluating-capabilitie](skills/timemachine-bench-benchmark-evaluating-capabilitie/SKILL.md) | Evaluation & Benchmarking, Agentic Systems | 143 |
| [tkg-thinker-dynamic-reasoning-over](skills/tkg-thinker-dynamic-reasoning-over/SKILL.md) | Reasoning & Chain-of-Thought | 244 |
| [tokenomics-quantifying-where-tokens](skills/tokenomics-quantifying-where-tokens/SKILL.md) | Multi-Agent Systems, Code & Software Engineering, Agentic Systems +2 | 227 |
| [toolself-unifying-task-execution](skills/toolself-unifying-task-execution/SKILL.md) | Agentic Systems, Data Processing | 217 |
| [toolweaver-weaving-collaborative-semantics](skills/toolweaver-weaving-collaborative-semantics/SKILL.md) | RAG & Retrieval, Data Processing | 181 |
| [topt-task-oriented-prompt-tuning](skills/topt-task-oriented-prompt-tuning/SKILL.md) | Prompt Engineering, Data Processing | 184 |
| [toward-cognitive-supersensing-multimodal](skills/toward-cognitive-supersensing-multimodal/SKILL.md) | Reasoning & Chain-of-Thought, Multimodal, Agentic Systems +1 | 173 |
| [toward-culturally-aligned-ontology-guided](skills/toward-culturally-aligned-ontology-guided/SKILL.md) | Multi-Agent Systems, Reasoning & Chain-of-Thought, Knowledge Graphs +2 | 190 |
| [toward-universal-transferable-jailbreak](skills/toward-universal-transferable-jailbreak/SKILL.md) | Security & Safety | 317 |
| [towards-adaptive-scalable-robust](skills/towards-adaptive-scalable-robust/SKILL.md) | Multi-Agent Systems, Evaluation & Benchmarking, Agentic Systems | 205 |
| [towards-agentic-intelligence-for](skills/towards-agentic-intelligence-for/SKILL.md) | Agentic Systems | 226 |
| [towards-ai-evaluation-domain-specific](skills/towards-ai-evaluation-domain-specific/SKILL.md) | RAG & Retrieval, Evaluation & Benchmarking, Data Processing | 260 |
| [towards-automated-kernel-generation](skills/towards-automated-kernel-generation/SKILL.md) | Agentic Systems, Efficiency & Optimization | 159 |
| [towards-autonomous-mathematics-research](skills/towards-autonomous-mathematics-research/SKILL.md) | RAG & Retrieval, Reasoning & Chain-of-Thought, Agentic Systems | 239 |
| [towards-declarative-agentic-layer](skills/towards-declarative-agentic-layer/SKILL.md) | Multi-Agent Systems, Agentic Systems | 218 |
| [towards-green-ai-decoding](skills/towards-green-ai-decoding/SKILL.md) | Code & Software Engineering, Efficiency & Optimization | 229 |
| [towards-holographic-characteristic-short-text](skills/towards-holographic-characteristic-short-text/SKILL.md) | Efficiency & Optimization, Data Processing | 150 |
| [towards-sample-efficient-stable-reinforcement](skills/towards-sample-efficient-stable-reinforcement/SKILL.md) | Efficiency & Optimization | 191 |
| [towards-science-collective-ai](skills/towards-science-collective-ai/SKILL.md) | Other | 184 |
| [towards-understanding-best-practices](skills/towards-understanding-best-practices/SKILL.md) | RAG & Retrieval, Memory & Context, Multimodal +2 | 189 |
| [tracecoder-trace-driven-multi-agent-framework](skills/tracecoder-trace-driven-multi-agent-framework/SKILL.md) | Multi-Agent Systems, Code & Software Engineering, Agentic Systems +1 | 191 |
| [tracellm-leveraging-prompt-engineering](skills/tracellm-leveraging-prompt-engineering/SKILL.md) | RAG & Retrieval, Prompt Engineering | 179 |
| [tracemem-weaving-narrative-memory](skills/tracemem-weaving-narrative-memory/SKILL.md) | Memory & Context, Agentic Systems, Data Processing | 229 |
| [tracenas-zero-shot-pruning-gradient](skills/tracenas-zero-shot-pruning-gradient/SKILL.md) | Fine-tuning & Training, Memory & Context, Prompt Engineering +1 | 226 |
| [trailblazer-history-guided-reinforcement-learning](skills/trailblazer-history-guided-reinforcement-learning/SKILL.md) | Security & Safety, Evaluation & Benchmarking, Memory & Context +2 | 244 |
| [training-data-selection-gradient](skills/training-data-selection-gradient/SKILL.md) | Fine-tuning & Training, Efficiency & Optimization, Data Processing +1 | 196 |
| [training-multi-turn-search-agent](skills/training-multi-turn-search-agent/SKILL.md) | RAG & Retrieval, Fine-tuning & Training, Multimodal +3 | 177 |
| [trapped-past-disentangling-fluid](skills/trapped-past-disentangling-fluid/SKILL.md) | Reasoning & Chain-of-Thought, Evaluation & Benchmarking, Fine-tuning & Training | 174 |
| [tre-encouraging-exploration-trust](skills/tre-encouraging-exploration-trust/SKILL.md) | RAG & Retrieval, Fine-tuning & Training | 204 |
| [treetensor-boost-ai-system](skills/treetensor-boost-ai-system/SKILL.md) | Other | 237 |
| [tricky2-benchmark-evaluating-human-error](skills/tricky2-benchmark-evaluating-human-error/SKILL.md) | Code & Software Engineering, Evaluation & Benchmarking | 207 |
| [trifuse-enhancing-attention-based-gui](skills/trifuse-enhancing-attention-based-gui/SKILL.md) | Fine-tuning & Training, Memory & Context, Multimodal +1 | 162 |
| [triplay-rl-tri-role-self-play-reinforcement](skills/triplay-rl-tri-role-self-play-reinforcement/SKILL.md) | Security & Safety, Evaluation & Benchmarking | 208 |
| [trust-design-skill-profiles](skills/trust-design-skill-profiles/SKILL.md) | Other | 215 |
| [trust-one-round-confidence](skills/trust-one-round-confidence/SKILL.md) | Data Processing | 209 |
| [ts-debate-multimodal-collaborative-debate](skills/ts-debate-multimodal-collaborative-debate/SKILL.md) | Multi-Agent Systems, Reasoning & Chain-of-Thought, Multimodal +3 | 232 |
| [tsaqa-time-series-analysis](skills/tsaqa-time-series-analysis/SKILL.md) | NLP & Text | 195 |
| [tsrbench-comprehensive-multi-task-multi-modal](skills/tsrbench-comprehensive-multi-task-multi-modal/SKILL.md) | Reasoning & Chain-of-Thought, Evaluation & Benchmarking, Multimodal +1 | 206 |
| [ttcs-test-time-curriculum-synthesis](skills/ttcs-test-time-curriculum-synthesis/SKILL.md) | Reasoning & Chain-of-Thought, Fine-tuning & Training, Data Processing | 226 |
| [tutorial-reasoning-ir-ir](skills/tutorial-reasoning-ir-ir/SKILL.md) | RAG & Retrieval, Reasoning & Chain-of-Thought, Evaluation & Benchmarking +1 | 249 |
| [twiff-think-future-frames](skills/twiff-think-future-frames/SKILL.md) | Other | 190 |
| [typhoon-s-minimal-open-post-training](skills/typhoon-s-minimal-open-post-training/SKILL.md) | Fine-tuning & Training | 214 |
| [ui-venus-15-technical-report](skills/ui-venus-15-technical-report/SKILL.md) | Multimodal, Agentic Systems, Data Processing | 252 |
| [uncertainty-and-fairness-awareness](skills/uncertainty-and-fairness-awareness/SKILL.md) | Evaluation & Benchmarking, Prompt Engineering | 230 |
| [understanding-agent-scaling-llm-based](skills/understanding-agent-scaling-llm-based/SKILL.md) | Multi-Agent Systems, Agentic Systems, Efficiency & Optimization | 204 |
| [understanding-dominant-themes-reviewing](skills/understanding-dominant-themes-reviewing/SKILL.md) | Other | 226 |
| [unicog-uncovering-cognitive-abilities](skills/unicog-uncovering-cognitive-abilities/SKILL.md) | Reasoning & Chain-of-Thought, Efficiency & Optimization | 178 |
| [unicomp-unified-evaluation-compression](skills/unicomp-unified-evaluation-compression/SKILL.md) | Evaluation & Benchmarking, Fine-tuning & Training, Efficiency & Optimization | 177 |
| [unifying-ranking-generation-query](skills/unifying-ranking-generation-query/SKILL.md) | RAG & Retrieval | 158 |
| [unikie-bench-benchmarking-multimodal-key](skills/unikie-bench-benchmarking-multimodal-key/SKILL.md) | Evaluation & Benchmarking, Multimodal, Prompt Engineering +2 | 292 |
| [unintended-memorization-sensitive-information](skills/unintended-memorization-sensitive-information/SKILL.md) | Fine-tuning & Training, Data Processing | 199 |
| [unit-based-agent-semi-cascaded-full-duplex](skills/unit-based-agent-semi-cascaded-full-duplex/SKILL.md) | Multimodal, Agentic Systems, Data Processing | 247 |
| [universal-anti-forensics-attack-against](skills/universal-anti-forensics-attack-against/SKILL.md) | Security & Safety, Multimodal | 254 |
| [unleashing-potential-sparse-attention](skills/unleashing-potential-sparse-attention/SKILL.md) | Memory & Context, Efficiency & Optimization | 321 |
| [unveiling-cognitive-compass-theory-of-mind-guided](skills/unveiling-cognitive-compass-theory-of-mind-guided/SKILL.md) | Reasoning & Chain-of-Thought, Evaluation & Benchmarking, Multimodal +2 | 197 |
| [urdubench-urdu-reasoning-benchmark](skills/urdubench-urdu-reasoning-benchmark/SKILL.md) | Reasoning & Chain-of-Thought, Evaluation & Benchmarking, Data Processing +1 | 171 |
| [usage-effects-requirements-ai-coding](skills/usage-effects-requirements-ai-coding/SKILL.md) | Code & Software Engineering, Prompt Engineering, Efficiency & Optimization | 228 |
| [use-graph-it-needs](skills/use-graph-it-needs/SKILL.md) | RAG & Retrieval, Evaluation & Benchmarking, Knowledge Graphs +2 | 254 |
| [v0-generalist-value-any-policy](skills/v0-generalist-value-any-policy/SKILL.md) | Fine-tuning & Training, Prompt Engineering, Efficiency & Optimization | 169 |
| [valueflow-measuring-propagation-value](skills/valueflow-measuring-propagation-value/SKILL.md) | Multi-Agent Systems, Evaluation & Benchmarking, Agentic Systems +1 | 180 |
| [variability-aware-detection-repair-compilation-err](skills/variability-aware-detection-repair-compilation-err/SKILL.md) | Code & Software Engineering | 180 |
| [vectra-metric-dataset-visual](skills/vectra-metric-dataset-visual/SKILL.md) | Evaluation & Benchmarking, Multimodal, Data Processing +1 | 305 |
| [veq-modality-adaptive-quantization-moe](skills/veq-modality-adaptive-quantization-moe/SKILL.md) | Memory & Context, Multimodal, Efficiency & Optimization +1 | 199 |
| [verge-formal-refinement-guidance](skills/verge-formal-refinement-guidance/SKILL.md) | Reasoning & Chain-of-Thought | 185 |
| [veri-sure-contract-aware-multi-agent-framework](skills/veri-sure-contract-aware-multi-agent-framework/SKILL.md) | Multi-Agent Systems, Reasoning & Chain-of-Thought, Agentic Systems | 186 |
| [vespo-variational-sequence-level-soft](skills/vespo-variational-sequence-level-soft/SKILL.md) | Evaluation & Benchmarking, Fine-tuning & Training, Efficiency & Optimization +1 | 189 |
| [vica-multimodal-vision-only-cross-attention](skills/vica-multimodal-vision-only-cross-attention/SKILL.md) | Memory & Context, Multimodal, Efficiency & Optimization | 232 |
| [videostf-stress-testing-output-repetition](skills/videostf-stress-testing-output-repetition/SKILL.md) | Security & Safety, Code & Software Engineering, Evaluation & Benchmarking +1 | 295 |
| [videothinker-building-agentic-videollms](skills/videothinker-building-agentic-videollms/SKILL.md) | RAG & Retrieval, Reasoning & Chain-of-Thought, Multimodal +2 | 204 |
| [vidvec-unlocking-video-mllm](skills/vidvec-unlocking-video-mllm/SKILL.md) | RAG & Retrieval, Fine-tuning & Training, Multimodal +3 | 197 |
| [vihermes-graph-grounded-multihop-question](skills/vihermes-graph-grounded-multihop-question/SKILL.md) | RAG & Retrieval, Reasoning & Chain-of-Thought, Knowledge Graphs +3 | 264 |
| [villain-at-averimatec-verifying](skills/villain-at-averimatec-verifying/SKILL.md) | Multi-Agent Systems, Reasoning & Chain-of-Thought, Multimodal +2 | 248 |
| [viola-video-in-context-learning](skills/viola-video-in-context-learning/SKILL.md) | RAG & Retrieval, Multimodal, Prompt Engineering +2 | 221 |
| [vision-deepresearch-benchmark-rethinking-visual-te](skills/vision-deepresearch-benchmark-rethinking-visual-te/SKILL.md) | RAG & Retrieval, Evaluation & Benchmarking, Multimodal +2 | 216 |
| [vision-deepresearch-incentivizing-deepresearch-cap](skills/vision-deepresearch-incentivizing-deepresearch-cap/SKILL.md) | RAG & Retrieval, Reasoning & Chain-of-Thought, Multimodal +2 | 180 |
| [vision-representations-artificial-intelligence](skills/vision-representations-artificial-intelligence/SKILL.md) | Security & Safety, Multimodal, Agentic Systems +1 | 281 |
| [visiontrim-unified-vision-token](skills/visiontrim-unified-vision-token/SKILL.md) | Fine-tuning & Training, Memory & Context, Multimodal +1 | 212 |
| [visor-visual-spatial-object](skills/visor-visual-spatial-object/SKILL.md) | Reasoning & Chain-of-Thought, Multimodal, Agentic Systems +2 | 198 |
| [vista-scene-aware-optimization-streaming](skills/vista-scene-aware-optimization-streaming/SKILL.md) | Efficiency & Optimization | 261 |
| [vistira-closing-image-text-modality](skills/vistira-closing-image-text-modality/SKILL.md) | Reasoning & Chain-of-Thought, Multimodal, Data Processing | 189 |
| [visual-cognitive-demands-model-powered](skills/visual-cognitive-demands-model-powered/SKILL.md) | Evaluation & Benchmarking, Multimodal, Agentic Systems | 290 |
| [visual-reasoning-over-time](skills/visual-reasoning-over-time/SKILL.md) | Multi-Agent Systems, Reasoning & Chain-of-Thought, Multimodal +2 | 180 |
| [vividface-real-time-realistic-facial](skills/vividface-real-time-realistic-facial/SKILL.md) | Multimodal, Data Processing | 277 |
| [vividvoice-unified-framework-scene-aware](skills/vividvoice-unified-framework-scene-aware/SKILL.md) | Memory & Context, Multimodal | 245 |
| [vlm-guided-iterative-refinement-surgical](skills/vlm-guided-iterative-refinement-surgical/SKILL.md) | RAG & Retrieval, Evaluation & Benchmarking, Multimodal +3 | 255 |
| [vln-pilot-vision-language-as-autonomous](skills/vln-pilot-vision-language-as-autonomous/SKILL.md) | Multimodal, Agentic Systems, Explainability | 235 |
| [vowelprompt-hearing-speech-emotions](skills/vowelprompt-hearing-speech-emotions/SKILL.md) | Multimodal, Prompt Engineering, Data Processing +1 | 287 |
| [voxmorph-scalable-zero-shot-voice](skills/voxmorph-scalable-zero-shot-voice/SKILL.md) | Evaluation & Benchmarking, Multimodal, Prompt Engineering +1 | 203 |
| [vtc-r1-vision-text-compression-long-context](skills/vtc-r1-vision-text-compression-long-context/SKILL.md) | Reasoning & Chain-of-Thought, Memory & Context, Multimodal +2 | 269 |
| [vulread-knowledge-graph-guided-software-vulnerabil](skills/vulread-knowledge-graph-guided-software-vulnerabil/SKILL.md) | Security & Safety, Code & Software Engineering, Reasoning & Chain-of-Thought +3 | 212 |
| [wavlink-compact-audio-text-embeddings](skills/wavlink-compact-audio-text-embeddings/SKILL.md) | RAG & Retrieval, Multimodal, Prompt Engineering | 197 |
| [wdscaling-parallel-tool-calling-deep](skills/wdscaling-parallel-tool-calling-deep/SKILL.md) | RAG & Retrieval, Reasoning & Chain-of-Thought, Agentic Systems | 161 |
| [weight-decay-improves-plasticity](skills/weight-decay-improves-plasticity/SKILL.md) | Fine-tuning & Training, Efficiency & Optimization | 169 |
| [what-makes-low-bit-quantization-aware](skills/what-makes-low-bit-quantization-aware/SKILL.md) | Reasoning & Chain-of-Thought, Fine-tuning & Training, Efficiency & Optimization +1 | 189 |
| [what-should-cite-rag](skills/what-should-cite-rag/SKILL.md) | RAG & Retrieval, Data Processing | 193 |
| [whats-benchmark-case-swe-bench-automated](skills/whats-benchmark-case-swe-bench-automated/SKILL.md) | Code & Software Engineering, Evaluation & Benchmarking | 318 |
| [when-agents-fail-act](skills/when-agents-fail-act/SKILL.md) | Agentic Systems | 194 |
| [when-agents-fail-comprehensive](skills/when-agents-fail-comprehensive/SKILL.md) | RAG & Retrieval, Code & Software Engineering, Memory & Context +1 | 240 |
| [when-agents-misremember-collectively](skills/when-agents-misremember-collectively/SKILL.md) | Multi-Agent Systems, Memory & Context, Agentic Systems +2 | 225 |
| [when-better-prompts-hurt](skills/when-better-prompts-hurt/SKILL.md) | RAG & Retrieval, Evaluation & Benchmarking, Agentic Systems +2 | 196 |
| [when-evaluation-becomes-side](skills/when-evaluation-becomes-side/SKILL.md) | Security & Safety, Evaluation & Benchmarking, Fine-tuning & Training +2 | 237 |
| [when-get-significantly-worse](skills/when-get-significantly-worse/SKILL.md) | Efficiency & Optimization | 245 |
| [when-iterative-rag-beats](skills/when-iterative-rag-beats/SKILL.md) | RAG & Retrieval, Reasoning & Chain-of-Thought, Data Processing +1 | 244 |
| [when-meets-fuzzy-topsis-personnel](skills/when-meets-fuzzy-topsis-personnel/SKILL.md) | Other | 233 |
| [when-much-imagine-adaptive](skills/when-much-imagine-adaptive/SKILL.md) | Multimodal, Agentic Systems, Data Processing | 228 |
| [when-should-search-more](skills/when-should-search-more/SKILL.md) | RAG & Retrieval | 300 |
| [when-silence-golden-learn](skills/when-silence-golden-learn/SKILL.md) | Other | 228 |
| [where-ai-coding-agents](skills/where-ai-coding-agents/SKILL.md) | Agentic Systems | 193 |
| [whispers-wealth-red-teaming-googles](skills/whispers-wealth-red-teaming-googles/SKILL.md) | Security & Safety, Agentic Systems, Prompt Engineering +1 | 218 |
| [white-box-sensitivity-auditing-steering](skills/white-box-sensitivity-auditing-steering/SKILL.md) | Other | 183 |
| [whitespaces-dont-lie-feature-driven](skills/whitespaces-dont-lie-feature-driven/SKILL.md) | Evaluation & Benchmarking | 255 |
| [who-deserves-reward-sharp](skills/who-deserves-reward-sharp/SKILL.md) | Multi-Agent Systems, Agentic Systems, Efficiency & Optimization +1 | 220 |
| [who-gets-which-message](skills/who-gets-which-message/SKILL.md) | Evaluation & Benchmarking, Data Processing | 221 |
| [why-ai-agents-systematically](skills/why-ai-agents-systematically/SKILL.md) | Agentic Systems | 309 |
| [why-attention-patterns-exist](skills/why-attention-patterns-exist/SKILL.md) | Memory & Context | 247 |
| [why-deep-research-agent](skills/why-deep-research-agent/SKILL.md) | RAG & Retrieval, Agentic Systems | 199 |
| [why-reasoning-fails-plan](skills/why-reasoning-fails-plan/SKILL.md) | Code & Software Engineering, Reasoning & Chain-of-Thought, Agentic Systems +1 | 218 |
| [why-steering-works-unified](skills/why-steering-works-unified/SKILL.md) | Other | 192 |
| [wideseek-r1-exploring-width-scaling](skills/wideseek-r1-exploring-width-scaling/SKILL.md) | RAG & Retrieval, Agentic Systems, Explainability +1 | 197 |
| [wiki-live-challenge-challenging](skills/wiki-live-challenge-challenging/SKILL.md) | RAG & Retrieval, Evaluation & Benchmarking, Agentic Systems +1 | 264 |
| [wildreward-learning-reward-in-the-wild](skills/wildreward-learning-reward-in-the-wild/SKILL.md) | Fine-tuning & Training, Data Processing | 175 |
| [will-it-survive-deciphering](skills/will-it-survive-deciphering/SKILL.md) | Evaluation & Benchmarking | 173 |
| [winflora-incentivizing-client-adaptive-aggregation](skills/winflora-incentivizing-client-adaptive-aggregation/SKILL.md) | Fine-tuning & Training | 198 |
| [world-workflows-benchmark-bringing](skills/world-workflows-benchmark-bringing/SKILL.md) | Evaluation & Benchmarking | 185 |
| [xai-clip-roi-guided-perturbation-framework](skills/xai-clip-roi-guided-perturbation-framework/SKILL.md) | Multimodal, Data Processing, Explainability +1 | 226 |
| [xlist-hate-checklist-based-framework-interpretable](skills/xlist-hate-checklist-based-framework-interpretable/SKILL.md) | Security & Safety, Multimodal, Efficiency & Optimization +3 | 229 |
| [yasa-scalable-multi-language-taint](skills/yasa-scalable-multi-language-taint/SKILL.md) | Security & Safety | 204 |
| [yoloe-26-integrating-yolo26-yoloe](skills/yoloe-26-integrating-yolo26-yoloe/SKILL.md) | Other | 232 |
| [your-secretly-contains-personality](skills/your-secretly-contains-personality/SKILL.md) | Efficiency & Optimization, Data Processing | 233 |
| [yunque-deepresearch-technical-report](skills/yunque-deepresearch-technical-report/SKILL.md) | Multi-Agent Systems, RAG & Retrieval, Agentic Systems | 191 |
| [zero-shot-product-attribute-labeling](skills/zero-shot-product-attribute-labeling/SKILL.md) | Evaluation & Benchmarking, Multimodal, Prompt Engineering +2 | 268 |
| [zero-sum-svd-balancing](skills/zero-sum-svd-balancing/SKILL.md) | Fine-tuning & Training, Efficiency & Optimization | 236 |
| [zero2text-zero-training-cross-domain-inversion](skills/zero2text-zero-training-cross-domain-inversion/SKILL.md) | RAG & Retrieval, Security & Safety, Evaluation & Benchmarking +3 | 188 |
| [zipmoe-on-device-moe-serving](skills/zipmoe-on-device-moe-serving/SKILL.md) | Efficiency & Optimization | 213 |
