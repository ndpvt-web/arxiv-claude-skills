---
name: "shieldedcode-learning-robust-representations"
description: |
  Apply ShieldedCode's protection-aware framework for virtual machine protected (VMP) code: generate VM bytecode from source, normalize and canonicalize VM instructions, model hierarchical instruction dependencies, rank protection effectiveness of VM variants, and detect binary similarity across obfuscation levels. Trigger phrases: "protect this code with VM obfuscation", "generate VMP bytecode", "normalize VM instructions", "rank VM protection strength", "detect similarity in obfuscated binaries", "reverse engineering resistance".
---

# ShieldedCode: Learning Robust Representations for VM-Protected Code

This skill enables Claude to apply the ShieldedCode framework (ICLR 2026) for reasoning about virtual machine protected (VMP) code. ShieldedCode is the first protection-aware approach that learns structured representations of VM-obfuscated binaries by modeling hierarchical instruction dependencies, jointly optimizing functionality-aware and protection-aware contrastive objectives, and ranking the effectiveness of different VM protection variants. Using these techniques, Claude can generate normalized VM bytecode from source, canonicalize disassembled VM output for analysis, assess protection strength across VM variants, and detect semantic similarity between obfuscated binaries.

## When to Use

- When the user asks to **generate VM-protected bytecode** from C/C++ source functions and wants structured, normalized output
- When the user needs to **normalize disassembled VM instructions** by replacing addresses with canonical labels, removing debug artifacts, and stabilizing tokenization
- When the user wants to **compare or rank VM protection variants** derived from the same source to determine which offers stronger obfuscation
- When the user asks to **detect functional similarity** between two binaries compiled at different optimization levels (O0-O3) or protection levels (L0-L3)
- When the user is building a **training pipeline for code protection models** and needs guidance on dataset construction, contrastive objectives, or hierarchical attention masking
- When the user wants to **assess reverse engineering resistance** of their protected code against pattern matching or symbolic execution attacks

## Key Technique

ShieldedCode's core insight is that VM-protected code has exploitable structure at three dependency levels that standard language models miss. Rather than treating VM bytecode as a flat token sequence, the framework inserts `[VINST]` marker tokens at instruction boundaries and defines a hierarchical attention mask: **intra-instruction** (tokens attend within their own virtual instruction plus its marker), **preceding-instruction** (tokens also attend to the immediately prior instruction's marker for short-range patterns), and **inter-instruction** (tokens attend to all prior markers for long-range control flow). This structured attention forces the model to learn instruction-level semantics rather than memorizing byte patterns.

The training objective combines standard language modeling loss with two contrastive terms. **Functionality-aware contrastive learning (FCL)** pulls embeddings of the same function across different protection levels together, weighted exponentially so adjacent levels (e.g., L0 vs L1) contribute more than distant ones. **Protection-aware contrastive learning (PCL)** enforces that embedding distances scale linearly with protection level differences, so L0-vs-L3 pairs are farther apart than L0-vs-L1 pairs. The combined loss is `L_vmp = L_lm + lambda * (L_fcl + L_pcl)`. This dual objective teaches the model both *what* a function does (semantic equivalence) and *how strongly* it is protected (protection strength ordering).

A separate **Protection Effectiveness Optimization (PEO)** task ranks VM variants using hard-negative contrastive learning. Given a query function, the model must identify the correct VM variant from a pool of K candidates (K up to 500), with difficult negatives weighted proportionally to their similarity rank. This enables quantitative comparison of protection schemes -- critical for selecting the strongest VM configuration for deployment.

## Step-by-Step Workflow

1. **Compile the source function** at the target optimization level (O0-O3) to produce a native binary: `exe = Compile(source.c, -O2)`. If working from existing binaries, skip to step 3.

2. **Apply VM protection** using a VMP tool (e.g., VMProtect, Themida, Code Virtualizer) at the desired protection level (L0-L3) to transform native instructions into virtual bytecode: `vm_binary = VMP(exe, level=L2)`.

3. **Disassemble the VM-protected binary** to extract raw virtual instruction sequences. Use a disassembler that can identify the VM dispatcher and handler table.

4. **Normalize the VM bytecode** using the four-step canonicalization process:
   - Strip all debug symbols, comments, and metadata
   - Insert whitespace around virtual instruction delimiters for stable tokenization
   - Replace virtual memory addresses with symbolic references (e.g., `0x4015a0` becomes `[ADDR_1]`)
   - Map instruction addresses to canonical labels: `[VINST-1]`, `[VINST-2]`, etc., in sequential order

5. **Insert hierarchical markers** by placing a `[VINST]_t` token at the boundary of each virtual instruction `t`. Define the attention mask so each token can see: (a) all tokens in its own instruction + its marker, (b) the preceding instruction's marker `[VINST]_{t-1}`, and (c) all prior markers for long-range flow.

6. **Extract embeddings** for comparison tasks: average the `[VINST]` marker embeddings for VM code representations, or average all token embeddings for source code representations.

7. **Apply contrastive ranking** to compare protection variants: compute pairwise cosine similarity between embeddings of the same function at different protection levels. Variants with greater embedding distance from the source indicate stronger protection.

8. **For code generation tasks**, use the structured prompt format: `"# This is the source code with {protection_level} protection: {source_code}"` and generate the normalized VM bytecode sequence.

9. **Validate generated VM code** using Pass@K evaluation: compile the generated bytecode, execute against test cases, and verify functional equivalence with the original source.

10. **Iterate on protection selection** by generating multiple VM variants and ranking them via PEO scoring to identify the configuration with maximum reverse engineering resistance.

## Concrete Examples

**Example 1: Normalizing Disassembled VM Bytecode**

User: "I have this disassembled VM bytecode from a protected binary. Can you normalize it for analysis?"

```
Raw disassembly:
  0x401000: vmov v0, [0x7ffc3a20]
  0x401008: vadd v1, v0, 0x5
  0x401010: vcmp v1, [0x7ffc3a28]
  0x401018: vjnz 0x401030
  0x401020: vmov [0x7ffc3a30], v1
  0x401028: vret
  0x401030: vsub v1, v1, 0x1
  0x401038: vjmp 0x401010
```

Approach:
1. Strip any debug annotations or comments
2. Replace absolute instruction addresses with canonical labels
3. Replace memory addresses with symbolic references
4. Insert whitespace around delimiters for tokenization stability

Output:
```
[VINST-1]: vmov v0 , [MEM_1]
[VINST-2]: vadd v1 , v0 , 0x5
[VINST-3]: vcmp v1 , [MEM_2]
[VINST-4]: vjnz [VINST-7]
[VINST-5]: vmov [MEM_3] , v1
[VINST-6]: vret
[VINST-7]: vsub v1 , v1 , 0x1
[VINST-8]: vjmp [VINST-3]
```

Note: Jump targets now reference canonical labels, memory addresses use symbolic names, and whitespace surrounds all delimiters.

**Example 2: Ranking VM Protection Variants**

User: "I have three VM protection variants of the same function. Which provides the strongest protection?"

```
Source (C):
int clamp(int x, int lo, int hi) {
    if (x < lo) return lo;
    if (x > hi) return hi;
    return x;
}

Variant A (L1 protection): 12 virtual instructions, direct register mapping
Variant B (L2 protection): 23 virtual instructions, split register mapping with dummy ops
Variant C (L3 protection): 41 virtual instructions, opaque predicates + handler duplication
```

Approach:
1. Normalize each variant using the canonicalization pipeline
2. Extract `[VINST]` marker embeddings for each variant and the source
3. Compute cosine distance from source embedding to each variant
4. Rank by distance: greater distance = stronger protection
5. Verify with PCL constraint: distances should scale roughly linearly with protection level

Output:
```
Protection Effectiveness Ranking:
  Variant C (L3): distance = 0.847 -- STRONGEST
  Variant B (L2): distance = 0.612
  Variant A (L1): distance = 0.391 -- WEAKEST

Recommendation: Variant C provides the strongest protection.
The ~0.23 distance gap between levels is consistent with
linear scaling (PCL property), confirming well-calibrated
protection levels.
```

**Example 3: Generating VM Bytecode from Source**

User: "Generate L0-level VM-protected bytecode for this function."

```c
int factorial(int n) {
    int result = 1;
    for (int i = 2; i <= n; i++)
        result *= i;
    return result;
}
```

Approach:
1. Parse the source function to identify control flow (loop, conditional, return)
2. Map C operations to virtual instruction equivalents
3. Assign virtual registers (v0 for n, v1 for result, v2 for i)
4. Apply L0 normalization (direct translation, no obfuscation padding)
5. Format with canonical labels and symbolic references

Output:
```
# This is the source code with L0 protection:
[VINST-1]:  vmov v1 , 0x1          ; result = 1
[VINST-2]:  vmov v2 , 0x2          ; i = 2
[VINST-3]:  vcmp v2 , v0           ; i <= n?
[VINST-4]:  vjgt [VINST-8]         ; if i > n, exit loop
[VINST-5]:  vmul v1 , v1 , v2      ; result *= i
[VINST-6]:  vadd v2 , v2 , 0x1     ; i++
[VINST-7]:  vjmp [VINST-3]         ; loop back
[VINST-8]:  vmov v0 , v1           ; return result
[VINST-9]:  vret
```

## Best Practices

- **Do:** Always normalize VM bytecode before any comparison or embedding extraction. Raw addresses and debug symbols create false dissimilarity between functionally equivalent code.
- **Do:** Use the hierarchical attention mask when building custom models. Flat attention over VM bytecode wastes capacity on irrelevant long-range token pairs while missing critical instruction-boundary structure.
- **Do:** Weight contrastive pairs by protection level adjacency (exponential decay for FCL). Adjacent levels share more structural similarity and provide stronger gradient signal than distant pairs.
- **Do:** Generate multiple VM variants and rank them via PEO before deploying protection. A single variant may have exploitable patterns that comparative ranking reveals.
- **Avoid:** Treating VM bytecode as natural language or standard assembly. Virtual instructions have custom semantics defined by the VM handler table -- standard disassembly heuristics will produce garbage.
- **Avoid:** Skipping the whitespace insertion step during normalization. Without stable token boundaries, subword tokenizers will split VM instructions inconsistently across samples, destroying alignment.

## Error Handling

| Problem | Cause | Solution |
|---------|-------|----------|
| Normalization produces inconsistent labels | Jump targets reference addresses not in the instruction list | Pre-scan all instruction addresses to build the complete label map before substitution |
| Embedding distances don't scale linearly | FCL/PCL weights are miscalibrated | Verify `tau_fcl` temperature and `beta` scaling factor; ensure training data covers all protection level pairs |
| Generated VM code fails functional tests | Model hallucinated virtual instructions not in the target VM's ISA | Constrain generation vocabulary to the specific VM's instruction set; post-validate against the handler table |
| Binary similarity detection returns false matches | Candidate pool contains functions with similar control flow structure | Increase hard-negative weight `kappa` in PEO and expand the candidate pool size K |
| Disassembly fails to identify VM dispatcher | Binary uses multi-layer or nested VM protection | Apply iterative peeling: identify and resolve the outermost VM layer first, then recurse |

## Limitations

- **VM-specific:** The normalization and hierarchical modeling assume a single-dispatch VM architecture. Nested VMs, JIT-compiled handlers, or hardware-assisted virtualization require architecture-specific adaptations.
- **No runtime semantics:** The framework operates on static bytecode. Dynamic behaviors like self-modifying handlers, runtime key derivation, or anti-debug checks are not captured in the representation.
- **Training data dependency:** Effective contrastive learning requires paired samples at multiple protection levels from the same source. If you only have a single protected binary with no source or variant access, the ranking and comparison capabilities are limited.
- **ISA coverage:** The paper's experiments focus on x86 VMP. ARM, MIPS, or RISC-V virtual machines may require retraining with architecture-specific normalization rules.
- **Not a replacement for formal verification:** Embedding-based protection ranking is a heuristic. It measures statistical resistance to learned attacks, not provable security guarantees.

## Reference

**Paper:** [ShieldedCode: Learning Robust Representations for Virtual Machine Protected Code](https://arxiv.org/abs/2601.20679v1) (ICLR 2026)
**Key sections:** Section 3 (VM normalization and hierarchical dependency modeling), Section 4 (FCL/PCL contrastive objectives and PEO task), Section 5 (two-stage training pipeline), Table 2 (Pass@1 results vs GPT-4o), Table 4 (binary similarity Recall@1 vs jTrans).