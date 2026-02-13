---
name: "modular-multi-task-learning-chemical"
description: |
  Build modular LoRA-based multi-task fine-tuning pipelines for chemical reaction prediction using LLMs on SMILES data.
  Implements parameter-efficient adapter strategies that prevent catastrophic forgetting across forward synthesis,
  retrosynthesis, and reagent prediction tasks. Use this skill when:
  - "Set up LoRA fine-tuning for a chemistry LLM"
  - "Build a multi-task reaction prediction pipeline"
  - "Fine-tune a model on SMILES data without forgetting"
  - "Create swappable adapters for different reaction types"
  - "Predict products, reagents, or retrosynthesis routes from SMILES"
  - "Adapt a general chemistry model to a specialized reaction domain"
---

# Modular Multi-Task Learning for Chemical Reaction Prediction

This skill enables Claude to build parameter-efficient fine-tuning pipelines for chemical reaction prediction using Low-Rank Adaptation (LoRA) on sequence-to-sequence LLMs. The core technique trains modular LoRA adapters on SMILES-encoded reaction data across three tasks — forward prediction, retrosynthesis, and reagent prediction — using task-specific prefixes. The approach achieves accuracy comparable to or exceeding full fine-tuning (78.3% vs 69.6% Acc@1 on C-H borylation) while retaining 40-80% of general chemistry performance where full fine-tuning collapses to <1%. Based on Pang et al. (2026), arXiv:2602.10404.

## When to Use

- When the user needs to fine-tune a chemistry LLM (ByT5, nach0, or similar T5-style model) on a domain-specific reaction dataset
- When building a multi-task system that must handle forward synthesis, retrosynthesis, and reagent prediction simultaneously
- When adapting a general chemistry model to a specialized reaction class (e.g., C-H functionalisation, cross-coupling) without destroying general knowledge
- When the user has a small domain-specific dataset (hundreds to low thousands of reactions) and needs to avoid overfitting
- When designing a modular adapter system where different LoRA modules can be hot-swapped for different reaction domains
- When evaluating whether LoRA or full fine-tuning is appropriate for a chemistry prediction task

## Key Technique

**Modular LoRA for Chemistry Multi-Task Learning.** The method applies Low-Rank Adaptation to encoder-decoder models (ByT5-small at 300M params, nach0 at 220M) that process reaction SMILES strings at the byte level. Instead of updating all model weights, LoRA injects small trainable rank-decomposition matrices (rank r=16, scaling alpha=32) into the frozen base model's weight matrices. This creates lightweight adapters (~0.5-2% of total parameters) that can be independently trained, stored, and swapped per reaction domain.

**Multi-Task via Prefix Routing.** Three chemical tasks share a single model but are distinguished by text prefixes prepended to the input SMILES: `Product:` for forward prediction (reactants.reagents -> product), `Reactants:` for retrosynthesis (product -> reactants), and `Reagents:` for reagent prediction (reactants.product -> reagents). The dot separator `.` joins multiple SMILES strings; the arrow `>>` separates input from output roles. Joint training across all three tasks in a single adapter produces a generalist module, while task-specific adapters can be trained separately for specialization.

**Catastrophic Forgetting Resistance.** The critical practical advantage: when fine-tuning on a small domain dataset (e.g., 557 C-H borylation reactions), full fine-tuning destroys the model's general USPTO knowledge (accuracy drops to <1%), while LoRA preserves 40-80%+ of original performance. This means a LoRA-adapted model can still handle general organic chemistry queries alongside its specialized domain — essential for production deployment where you cannot maintain separate model copies per reaction class.

## Step-by-Step Workflow

1. **Prepare the base model.** Select a byte-level or subword chemistry LLM as the foundation. ByT5-small (300M params) is the validated choice; nach0-base (220M) also works. Trim the tokenizer vocabulary to only tokens appearing in SMILES strings to reduce memory and improve convergence.

2. **Format reaction data as prefix-tagged SMILES strings.** Convert each reaction into three task views:
   - Forward: `Product: CC(=O)O.CCO>>` → `CCOC(C)=O`
   - Retrosynthesis: `Reactants: CCOC(C)=O>>` → `CC(=O)O.CCO`
   - Reagent: `Reagents: CC(=O)O.CCOC(C)=O>>` → `[H+]`
   Canonicalize all SMILES with RDKit before formatting. Split data 80/10/10 train/val/test.

3. **Configure LoRA hyperparameters.** Set rank r=16, alpha=32 (scaling factor 2.0), dropout=0.0. Apply LoRA to all linear layers in the encoder and decoder (query, key, value, and output projections). These settings are validated across multiple reaction types; lower ranks (r=4, r=8) work for simpler datasets but underperform on complex reactions.

4. **Set up the training loop.** Use learning rate 0.003, train for 50-100 epochs with early stopping on validation loss. For multi-task training, interleave examples from all three tasks in each batch. For domain-specific adaptation (e.g., C-H borylation), first train a general multi-task adapter on USPTO, then further fine-tune a copy on the domain data.

5. **Train task-specific vs. multi-task adapters.** Decide between: (a) a single multi-task adapter handling all three tasks (simpler, good baseline), or (b) separate adapters per task (higher peak performance on individual tasks). For the multi-task adapter, sample tasks proportionally to dataset size. Save adapter checkpoints separately from the base model.

6. **Evaluate with Accuracy@K (K=1,3,5).** Generate top-K predictions using beam search. Validate SMILES syntax with RDKit — invalid SMILES count as incorrect. Report Acc@1 as the primary metric. Use Cliff's delta effect size and Wilcoxon signed-rank test to determine if differences between LoRA and full fine-tuning are statistically significant.

7. **Measure catastrophic forgetting.** After domain fine-tuning, re-evaluate the adapter on the original general dataset (e.g., USPTO_1K_TPL). Compare: full fine-tuning typically collapses to <1% on the general task; LoRA should retain 40-80%+. This retention metric is as important as domain accuracy for production viability.

8. **Implement adapter hot-swapping for inference.** Store each trained LoRA adapter as a small file (~2-10 MB). At inference time, load the frozen base model once, then dynamically attach the appropriate LoRA adapter based on the reaction domain or task. Use PEFT's `set_adapter()` or equivalent to switch between adapters without reloading the base model.

9. **Validate generalization with alternative predictions.** Check that the model produces chemically plausible alternatives beyond exact matches (e.g., predicting valid alternative solvents). Manual expert review or RDKit property checks on top-K predictions beyond the ground truth provide evidence of genuine chemical reasoning vs. memorization.

## Concrete Examples

**Example 1: Setting Up a LoRA Fine-Tuning Pipeline for Reaction Prediction**

User: "I have 700 C-H borylation reactions in a CSV with columns [reactants, reagents, product, yield]. Help me fine-tune ByT5-small to predict products."

Approach:
1. Filter reactions to yield > 0.3 to remove low-confidence data (~685 remaining)
2. Format as SMILES strings: `Product: {reactants}.{reagents}>>`  → `{product}`
3. Split 80/10/10, canonicalize with RDKit
4. Configure LoRA and train

Output (Python with HuggingFace PEFT):
```python
import pandas as pd
from rdkit import Chem
from peft import LoraConfig, get_peft_model, TaskType
from transformers import AutoTokenizer, AutoModelForSeq2SeqLM, Seq2SeqTrainer

# Load and filter data
df = pd.read_csv("ch_borylation.csv")
df = df[df["yield"] > 0.3]

# Canonicalize SMILES
def canon(smi):
    mol = Chem.MolFromSmiles(smi)
    return Chem.MolToSmiles(mol) if mol else None

df["reactants"] = df["reactants"].apply(canon)
df["product"] = df["product"].apply(canon)
df = df.dropna(subset=["reactants", "product"])

# Format for forward prediction
df["input_text"] = "Product: " + df["reactants"] + "." + df["reagents"] + ">>"
df["target_text"] = df["product"]

# Load base model
model_name = "google/byt5-small"
tokenizer = AutoTokenizer.from_pretrained(model_name)
model = AutoModelForSeq2SeqLM.from_pretrained(model_name)

# Configure LoRA: r=16, alpha=32, no dropout
lora_config = LoraConfig(
    task_type=TaskType.SEQ_2_SEQ_LM,
    r=16,
    lora_alpha=32,
    lora_dropout=0.0,
    target_modules=["q", "k", "v", "o", "wi", "wo"],
)
model = get_peft_model(model, lora_config)
model.print_trainable_parameters()  # ~0.5-2% of total

# Train with lr=0.003, 50 epochs, early stopping
# ... standard Seq2SeqTrainer setup ...
```

**Example 2: Multi-Task Adapter with Three Reaction Tasks**

User: "I need one model that handles forward prediction, retrosynthesis, and reagent prediction. How do I set up the multi-task training?"

Approach:
1. Create three views of every reaction in the dataset
2. Prepend task-specific prefixes
3. Train a single LoRA adapter on the interleaved tasks

Output (data formatting):
```python
def create_multitask_examples(row):
    """Generate three task views from one reaction record."""
    r, g, p = row["reactants"], row["reagents"], row["product"]
    return [
        {"input": f"Product: {r}.{g}>>",    "target": p},     # forward
        {"input": f"Reactants: {p}>>",       "target": f"{r}.{g}"},  # retro
        {"input": f"Reagents: {r}.{p}>>",    "target": g},     # reagent
    ]

# Interleave all tasks into training set
all_examples = []
for _, row in df.iterrows():
    all_examples.extend(create_multitask_examples(row))

# Shuffle and train with same LoRA config (r=16, alpha=32)
import random
random.shuffle(all_examples)
```

Expected results: Multi-task LoRA achieves ~75% Acc@1 on forward and ~74% on reagent prediction for C-H borylation, vs. ~56% and ~55% for full fine-tuning multi-task.

**Example 3: Adapter Hot-Swapping for Multiple Reaction Domains**

User: "I have LoRA adapters for USPTO general reactions and C-H borylation. How do I switch between them at inference without reloading the base model?"

Approach:
1. Load base model once
2. Load both adapters
3. Switch dynamically per query

Output:
```python
from peft import PeftModel

# Load base model once
base_model = AutoModelForSeq2SeqLM.from_pretrained("google/byt5-small")

# Load adapters (each is ~5MB on disk)
model = PeftModel.from_pretrained(base_model, "adapters/uspto_general", adapter_name="uspto")
model.load_adapter("adapters/ch_borylation", adapter_name="ch_borylation")

# Route query to appropriate adapter
def predict_reaction(smiles_input, domain="general"):
    adapter = "ch_borylation" if domain == "ch_borylation" else "uspto"
    model.set_adapter(adapter)
    inputs = tokenizer(smiles_input, return_tensors="pt")
    outputs = model.generate(**inputs, num_beams=5, num_return_sequences=5)
    return [tokenizer.decode(o, skip_special_tokens=True) for o in outputs]

# Forward prediction with domain-specific adapter
products = predict_reaction("Product: c1ccccc1.B2OC(C)(C)C(C)(C)O2>>", domain="ch_borylation")
# Returns top-5 predicted product SMILES
```

## Best Practices

- **Do:** Canonicalize all SMILES with RDKit before training and evaluation. Non-canonical SMILES create false negatives in accuracy metrics.
- **Do:** Always measure catastrophic forgetting by evaluating on the original general dataset after domain fine-tuning. A model with 78% domain accuracy but 0% general accuracy is not production-ready.
- **Do:** Use beam search with K=5 and report Acc@1 through Acc@5. Top-1 misses may still be chemically valid alternatives (e.g., equivalent solvents).
- **Do:** Keep LoRA dropout at 0.0 for small chemistry datasets (<1000 reactions). Dropout destabilizes training at this scale.
- **Avoid:** Using subword tokenizers that fragment SMILES tokens unpredictably. Byte-level models (ByT5) handle SMILES character structure natively.
- **Avoid:** Training separate base models per task when adapters suffice. The whole point of modular LoRA is one base model, many lightweight adapters.

## Error Handling

- **Invalid SMILES output:** The model may generate syntactically invalid SMILES. Always validate outputs with `Chem.MolFromSmiles()`. If invalid, fall back to the next beam search candidate. Report the invalid SMILES rate as a quality metric.
- **Training divergence on small datasets:** With <500 reactions, LoRA training can be unstable. Reduce learning rate to 0.001, increase epochs to 100, and use gradient clipping. If divergence persists, reduce rank to r=8.
- **Task confusion in multi-task mode:** If the model produces reagents when asked for products, ensure task prefixes (`Product:`, `Reagents:`, `Reactants:`) are consistently formatted with no trailing spaces or case variations.
- **Low retrosynthesis accuracy:** Retrosynthesis is inherently harder (one-to-many mapping). Expect 5-15% lower Acc@1 than forward prediction. Increase beam width and evaluate Acc@5 for a fairer comparison.
- **Adapter incompatibility:** LoRA adapters are tied to the base model architecture. An adapter trained on ByT5-small cannot be loaded onto ByT5-base. Always version-lock the base model checkpoint with each adapter.

## Limitations

- Validated only on ByT5-small (300M) and nach0-base (220M). Scaling behavior to larger models (1B+) is hypothesized but not demonstrated.
- SMILES is a string representation that doesn't encode 3D geometry. Reactions sensitive to stereochemistry or conformational effects may be poorly predicted.
- The smallest tested domain dataset had 557 training reactions. Below ~200 reactions, even LoRA may overfit without additional regularization or data augmentation.
- Multi-task training assumes balanced task difficulty. If one task dominates (e.g., forward prediction is much easier), it can starve learning signal from harder tasks. Task-weighted sampling may be needed.
- The approach does not model reaction conditions (temperature, pressure, time) — only molecular identity. Yield prediction requires a separate regression head.

## Reference

**Paper:** Pang, Zaitoun, Couso Cambeiro, Vulić. "Modular Multi-Task Learning for Chemical Reaction Prediction." arXiv:2602.10404v1, 2026.
**Key insight:** LoRA adapters on byte-level chemistry LLMs match or exceed full fine-tuning accuracy while preserving 40-80% of general chemistry knowledge (vs. <1% for full fine-tuning), enabling modular deployment of domain-specific reaction predictors from a single base model.