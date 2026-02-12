---
name: "steuerllm-local-specialized-german"
description: "Build domain-specialized LLM pipelines for formal-rule domains (tax law, legal, regulatory) using retrieval-augmented synthetic data generation and statement-level evaluation. Triggers: 'domain-adapt a model for legal text', 'build a tax law QA system', 'create a benchmark for regulatory compliance', 'generate synthetic legal training data', 'evaluate LLM on statute-based reasoning', 'fine-tune for German tax law'"
---

# Domain-Specialized LLM Pipeline for Formal-Rule Domains

This skill teaches Claude to build end-to-end domain-specialized LLM systems following the SteuerLLM methodology: a controlled retrieval-augmented pipeline that generates high-quality synthetic training data from authentic domain material, paired with a rigorous statement-level partial-credit evaluation framework. The core insight is that domain-specific data curation and targeted architectural adaptation (block expansion) beat raw parameter scale for domains governed by strict formal rules, precise terminology, and structured reasoning — such as tax law, regulatory compliance, and statutory interpretation.

## When to Use

- When the user needs to fine-tune or domain-adapt an LLM for a formal-rule domain (tax, legal, medical regulation, financial compliance)
- When building a retrieval-augmented data generation pipeline to create synthetic QA pairs from legislation, statutes, or regulatory text
- When designing a benchmark with statement-level partial-credit scoring for evaluating legal or regulatory reasoning
- When the user asks to expand a base model's capacity via block expansion (e.g., Mistral 24B → 28B) before domain fine-tuning
- When creating an instruction-tuning dataset from structured legal documents using a "Water Fountain" generation algorithm
- When evaluating LLM outputs against rigid grading schemes requiring exact statutory citation and numerical accuracy

## Key Technique

**Controlled Retrieval-Augmented Synthetic Data Generation.** SteuerLLM's training pipeline generates instruction-tuning data by combining three components: (1) a domain-specific filtering pipeline that classifies and extracts tax-relevant content from web corpora, (2) a semantic search layer (SearXNG + embedding server) that retrieves relevant statutory passages for each training example, and (3) a "Water Fountain Algorithm" that synthesizes multi-turn QA conversations grounded in retrieved legislation. The pipeline produces two dataset types — a pretraining corpus for continued pretraining and a 277K-conversation instruction dataset covering income tax (EStG), corporate tax (KStG), VAT (UStG), inheritance tax (ErbStG), trade tax (GewStG), and procedural tax law (AO).

**Block Expansion + Two-Stage Training.** Rather than training from scratch, the approach expands a Mistral Small base model from 24B to 28B parameters by duplicating transformer blocks, then applies two-stage training: (1) continual pretraining on the domain corpus to internalize statutory language and structure, followed by (2) instruction fine-tuning on the synthetic QA dataset using Axolotl with DeepSpeed ZeRO-3 on H100 clusters. This achieves stronger performance than general-purpose models with 2-4x more parameters.

**Statement-Level Partial-Credit Evaluation.** The SteuerEx benchmark decomposes each exam question into individual scoreable statements, awarding partial credit for correct statutory citations, correct legal reasoning steps, and accurate numerical calculations independently. This mirrors real German tax law examination grading and provides fine-grained signal about where models fail — far more informative than binary pass/fail evaluation.

## Step-by-Step Workflow

1. **Define domain scope and source material.** Identify the formal-rule domain (e.g., German tax law) and collect authentic source texts: legislation (EStG, KStG, AO, UStG, ErbStG, GewStG), examination papers, court rulings, and official commentary. Organize by sub-domain.

2. **Build the domain-specific filtering pipeline.** Implement a multi-worker document classifier that scores web-crawled or corpus documents for domain relevance. Use keyword matching on statutory references (e.g., "§ 4 EStG", "Abgabenordnung") combined with a trained text classifier to filter at scale. Output a clean pretraining corpus.

3. **Set up the retrieval infrastructure.** Deploy a semantic search stack: SearXNG instance (port 8080) for broad retrieval, an embedding server (port 8000) for dense passage retrieval over indexed statutes, and a tokenization service (port 8001). Index all statutory texts, commentary, and case law as retrievable passages.

4. **Implement the Water Fountain generation algorithm.** For each source document (exam question, case study, statutory section), retrieve the top-k relevant statutory passages, then prompt a teacher LLM to generate multi-turn QA conversations grounded in those passages. Enforce that every assistant response must cite specific statute paragraphs (e.g., "§ 15 Abs. 1 Satz 1 Nr. 1 EStG") and show calculation steps where applicable. Generate both standard QA pairs (~215K) and diversity-augmented pairs (~62K) to improve robustness.

5. **Expand the base model architecture.** Take a strong instruction-tuned base (e.g., Mistral Small 24B) and apply block expansion — duplicate selected transformer layers to increase capacity to 28B parameters. Initialize duplicated blocks from existing weights to preserve learned representations.

6. **Run continual pretraining.** Train the expanded model on the filtered domain corpus using standard causal language modeling. This stage internalizes statutory language patterns, section numbering conventions, and domain-specific terminology without forgetting general capabilities.

7. **Instruction fine-tune on synthetic QA data.** Fine-tune using the 277K-conversation instruction dataset with Axolotl framework, DeepSpeed ZeRO-3 optimization, and a low temperature (0.3 recommended at inference). Format training data as multi-turn conversations with `user`/`assistant` roles.

8. **Build the statement-level evaluation benchmark.** Decompose each test question into individual scoreable statements. Define a rubric that awards partial credit for: (a) correct identification of applicable statute, (b) correct legal subsumption (applying law to facts), (c) accurate numerical computation, and (d) proper structural argumentation. Assign point values (typically 1-18.5 per question).

9. **Evaluate with bootstrap confidence intervals.** Run the model on benchmark questions, score each statement-level response against the rubric, and compute aggregate scores with bootstrap resampling for statistical reliability. Compare against general-purpose baselines at equivalent and larger scales.

10. **Deploy with constrained generation settings.** Serve the model with temperature=0.3, include the chat template from the model card, and wrap responses in a structured format that separates statutory citations, legal reasoning, and numerical calculations.

## Concrete Examples

**Example 1: Building a synthetic QA dataset from tax legislation**

```
User: I have a collection of German tax law statutes (EStG, KStG, UStG) as plain text files.
I need to generate a training dataset for fine-tuning an LLM on tax law QA.

Approach:
1. Index all statute files into a vector store using sentence-transformers embeddings
   (e.g., intfloat/multilingual-e5-large), chunked at section level (§).
2. For each statute section, retrieve 3-5 related sections from other laws that
   interact with it (e.g., § 8 KStG references § 4-7 EStG for profit calculation).
3. Prompt a teacher model (e.g., GPT-4 or Claude) with this template:

   SYSTEM: You are a German tax law examiner. Generate a realistic exam-style question
   and detailed model answer based on the following statutory provisions. The answer MUST:
   - Cite specific paragraphs (§ X Abs. Y Satz Z [Law])
   - Show all calculation steps with intermediate results
   - Apply the legal subsumption method (Obersatz, Untersatz, Ergebnis)

   CONTEXT: {retrieved_statute_sections}
   TOPIC: {primary_section_title}

4. Generate ~2000 QA pairs per tax domain, then create diversity variants by:
   - Changing numerical values in fact patterns
   - Swapping taxpayer circumstances (married/single, employed/self-employed)
   - Adding edge cases (international elements, special provisions)

Output (one generated conversation):
{
  "id": "estg-15-synth-0042",
  "conversation": [
    {
      "role": "user",
      "content": "Herr Müller betreibt als Einzelunternehmer einen Handwerksbetrieb.
       Im Wirtschaftsjahr 2024 erzielt er Umsatzerlöse von 380.000 €, Materialaufwand
       von 95.000 € und Personalkosten von 120.000 €. Er hat einen PKW im
       Betriebsvermögen (Listenpreis 45.000 €), den er zu 30% privat nutzt.
       Ermitteln Sie den Gewinn aus Gewerbebetrieb."
    },
    {
      "role": "assistant",
      "content": "I. Einkünfte aus Gewerbebetrieb (§ 15 Abs. 1 Satz 1 Nr. 1 EStG)\n\n
       Herr Müller erzielt als Einzelunternehmer Einkünfte aus Gewerbebetrieb gem.
       § 15 Abs. 1 S. 1 Nr. 1 EStG...\n\n
       II. Gewinnermittlung (§ 4 Abs. 1 EStG)\n\n
       Umsatzerlöse: 380.000 €\n
       - Materialaufwand: -95.000 €\n
       - Personalkosten: -120.000 €\n
       = Vorläufiger Gewinn: 165.000 €\n\n
       III. Privatnutzung PKW (§ 6 Abs. 1 Nr. 4 Satz 2 EStG)\n\n
       1%-Regelung: 45.000 € × 1% × 12 = 5.400 €\n
       Entnahme für private Nutzung: +5.400 €\n\n
       Gewinn aus Gewerbebetrieb: 170.400 €"
    }
  ]
}
```

**Example 2: Designing a statement-level evaluation rubric**

```
User: I have an LLM that answers German tax law questions. How do I evaluate it
with partial-credit scoring like the SteuerEx benchmark?

Approach:
1. Decompose each exam question into atomic scoreable statements.
2. For each statement, define the expected content and point value.
3. Evaluate model output against each statement independently.

Example question (Einkommensteuer, max 8 points):
"Bestimmen Sie die Einkünfte von Frau Schmidt aus nichtselbständiger Arbeit."

Statement-level rubric:
| # | Expected Statement                                    | Points |
|---|-------------------------------------------------------|--------|
| 1 | Identifies § 19 Abs. 1 Satz 1 Nr. 1 EStG            | 1.0    |
| 2 | Correctly classifies gross salary as employment income | 1.0    |
| 3 | Identifies employer pension contribution (§ 3 Nr. 62) | 1.5    |
| 4 | Correctly computes Werbungskosten (§ 9 EStG)         | 1.5    |
| 5 | Applies Arbeitnehmer-Pauschbetrag (§ 9a Nr. 1 EStG) | 1.0    |
| 6 | Correct final numerical result                        | 2.0    |

Scoring implementation:
- Use an LLM-as-judge or keyword matching to check each statement
- Award full points only for exact statutory citation + correct application
- Award half points for correct reasoning with wrong citation
- Sum partial credits for the final score
- Report per-domain breakdown (EStG, KStG, UStG, AO, ErbStG, GewStG)
```

**Example 3: Block expansion for domain adaptation**

```
User: I want to expand Mistral Small 24B to 28B parameters before domain fine-tuning.
How does the block expansion method work?

Approach:
1. Load the base model and identify its transformer block structure.
2. Select blocks to duplicate (typically from the middle layers where
   representations are most general).
3. Copy weights from selected blocks to create new layers.
4. Insert duplicated blocks at their original positions, increasing depth.

Implementation sketch (PyTorch):
import torch
from transformers import AutoModelForCausalLM

model = AutoModelForCausalLM.from_pretrained("mistralai/Mistral-Small-24B-Instruct-2501")

# Identify layers to duplicate (e.g., layers 20-23 in a 32-layer model)
layers_to_duplicate = list(range(20, 24))

# Deep copy and insert
new_layers = torch.nn.ModuleList()
for i, layer in enumerate(model.model.layers):
    new_layers.append(layer)
    if i in layers_to_duplicate:
        new_layers.append(copy.deepcopy(layer))

model.model.layers = new_layers
# Model is now ~28B parameters with 36 layers

# Proceed with continual pretraining on domain corpus
# then instruction fine-tuning on synthetic QA data
```

## Best Practices

- **Do:** Always ground synthetic training data in retrieved statutory passages — never let the teacher LLM generate legal citations from memory, as hallucinated statute references poison the training set.
- **Do:** Use the legal subsumption structure (Obersatz → Definition → Untersatz → Ergebnis) as the required output format for training data, since this mirrors how German legal reasoning is formally structured and graded.
- **Do:** Generate diversity variants of training examples by systematically varying numerical values, taxpayer circumstances, and edge cases to prevent overfitting to specific fact patterns.
- **Do:** Evaluate with statement-level granularity rather than question-level binary scoring — a model that gets 6/8 statements right is meaningfully better than one that gets 3/8, and question-level scoring loses this signal.
- **Avoid:** Training on raw statutory text alone without QA formatting — statutes are written in legislative style that does not teach the model how to apply law to fact patterns.
- **Avoid:** Using temperature > 0.5 at inference for legal reasoning tasks — higher temperatures increase hallucinated citations and numerical errors. The paper recommends 0.3.

## Error Handling

- **Hallucinated statute citations:** If the model cites non-existent paragraphs (e.g., "§ 99 EStG" when EStG only has ~100 sections), implement a post-processing validator that checks cited statutes against an indexed reference database. Flag or suppress responses with invalid citations.
- **Numerical calculation errors:** Implement a deterministic calculator fallback for tax computations. Extract numerical steps from the model's chain-of-thought, verify each arithmetic operation, and correct or flag discrepancies.
- **Cross-domain confusion:** Models may apply the wrong tax law to a scenario (e.g., applying income tax rules to a corporate tax question). Include domain-routing logic that classifies the question into one of the six domains before prompting the model with domain-specific context.
- **Outdated law references:** Tax law changes annually. Version-stamp all training data and retrieval indices with the applicable tax year. At inference, verify the user's target year matches the model's training vintage.

## Limitations

- This approach is specifically validated for German tax law. Transferring to other legal systems (common law, different civil law jurisdictions) requires re-building the statutory retrieval index and re-generating training data — the methodology transfers, but the data does not.
- The 28B parameter model requires significant GPU resources (H100 cluster for training, single A100/H100 for inference). Smaller deployments need quantization (GPTQ/AWQ), which may degrade precision on numerical calculations.
- Statement-level evaluation requires expert-crafted rubrics for each question, which is labor-intensive. Automated rubric generation with LLM-as-judge introduces its own error modes.
- The synthetic data pipeline depends on a strong teacher model for QA generation. Teacher model errors propagate into training data — manual quality review of a representative sample (5-10%) is essential.
- Legal reasoning requires up-to-date statutory knowledge. The model's training cutoff creates a hard boundary — legislative changes after the training data vintage produce incorrect answers without RAG augmentation at inference time.

## Reference

**Paper:** [SteuerLLM: Local specialized large language model for German tax law analysis](https://arxiv.org/abs/2602.11081v1) — Look for: the Water Fountain Algorithm for synthetic data generation (Section 3), the block expansion methodology (Section 4), and the statement-level partial-credit evaluation framework in SteuerEx (Section 5). Code and data: [github.com/windprak/steuerllm](https://github.com/windprak/steuerllm).