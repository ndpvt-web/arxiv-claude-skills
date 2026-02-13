---
name: "helios-hierarchical-graph-abstraction"
description: "Structure-aware binary decompilation using hierarchical control-flow graph abstraction for LLMs. Converts binary programs into compilable, semantically faithful C code by encoding CFG structure (basic blocks, successors, loops, conditionals) as a hierarchical text prompt alongside raw decompiler output, with optional compiler-in-the-loop feedback. Trigger phrases: 'decompile this binary', 'reverse engineer this executable', 'convert assembly to C', 'decompile with control flow', 'structure-aware decompilation', 'recompilable decompilation'"
---

# HELIOS: Hierarchical Graph Abstraction for Structure-Aware LLM Decompilation

This skill enables Claude to perform high-quality binary decompilation by treating it as a structured reasoning task rather than plain text translation. Instead of feeding raw decompiler output directly to the LLM, HELIOS encodes the program's control-flow graph (CFG) and function-call graph (FCG) into a hierarchical text representation with three abstraction levels -- function-level metadata, block-level control flow, and instruction-level semantics -- then combines this with the raw pseudo-C output and optional compiler error feedback. This approach raises compilability from ~45-71% to 85-94%+ and improves functional correctness by up to 5.6 percentage points across x86, ARM, and MIPS architectures, all without model fine-tuning.

## When to Use

- When the user provides a binary, object file, or disassembly and asks you to produce compilable C source code
- When decompiler output (from Ghidra, IDA, Binary Ninja) contains broken control flow, missing branches, or won't compile
- When reverse engineering an optimized binary where the decompiler's pseudo-C is syntactically fragile or logically inconsistent
- When the user needs to decompile across architectures (x86_64, ARM, MIPS) and wants consistent results
- When analyzing malware or firmware and the user needs recompilable, semantically faithful C code for further analysis
- When the user has Ghidra P-Code or CFG data and wants to use it to improve decompilation quality

## Key Technique

**The core insight:** LLMs treat decompiled code as flat text and ignore the graph structures that govern program control flow. This causes them to hallucinate branches, invent variables, and produce code that doesn't compile. HELIOS fixes this by extracting the control-flow graph from a binary analysis tool (Ghidra), encoding it as structured text at three hierarchical levels, and supplying it alongside the raw decompiler output as a structured prompt.

**The three-level hierarchy:** Level 1 (FUNCTION_CONTEXT) provides a compact summary -- function name, signature, target architecture, number of basic blocks, and loop count. Level 2 (CFG_OVERVIEW) encodes the CFG as an adjacency list, showing each basic block's successors and structural role (loop header, branch target, exit block). Level 3 (BLOCK_DETAILS) contains distilled P-Code instructions for each basic block, giving the LLM ground-truth semantic operations to reason against. This hierarchy lets the LLM check its own output: every branch in the generated C must correspond to an edge in the CFG, and no new global variables should appear unless grounded in the block details or raw decompilation.

**Compiler-in-the-loop (CITL):** After the first generation pass, the output is compiled. If compilation fails, the compiler's error diagnostics are appended to the prompt in a dedicated section and the model is asked to correct the code while preserving control-flow consistency. This single feedback iteration pushes compilability above 94%.

## Step-by-Step Workflow

1. **Extract the CFG and metadata from the binary.** Use Ghidra (preferred), angr, or Binary Ninja to obtain: (a) the function signature and name, (b) the complete control-flow graph with basic blocks and directed edges, (c) P-Code or intermediate representation for each basic block, and (d) the interprocedural function-call graph restricted to calls from the target function.

2. **Generate raw decompiler pseudo-C.** Run Ghidra's decompiler (or equivalent) to produce the baseline pseudo-C output. Keep this unmodified -- it serves as the `[RAW_DECOMPILED_CODE]` section.

3. **Build the FUNCTION_CONTEXT section.** Write a concise header containing: function name, signature (return type and parameters), target architecture (e.g., x86_64, ARM32, MIPS), total number of basic blocks, and number of detected loops.

4. **Build the CFG_OVERVIEW section.** Encode the CFG as a text adjacency list. For each basic block, list its ID, successor block IDs, and structural role (e.g., `loop_header`, `branch_target`, `exit`). Example format:
   ```
   Block_0 (entry) -> [Block_1, Block_2]
   Block_1 (loop_header) -> [Block_3, Block_4]
   Block_2 (branch_target) -> [Block_5]
   ...
   ```

5. **Build the BLOCK_DETAILS section.** For each basic block, include a distilled summary of its P-Code operations -- assignments, comparisons, calls, memory accesses -- using stable block identifiers that match the CFG_OVERVIEW.

6. **Compose the structured prompt.** Assemble the four sections in order: `[FUNCTION_CONTEXT]`, `[CFG_OVERVIEW]`, `[BLOCK_DETAILS]`, `[RAW_DECOMPILED_CODE]`. Prepend a task description and critical reasoning rules:
   - All branches in generated code must correspond to edges in `[CFG_OVERVIEW]`
   - No new global variables unless present in `[BLOCK_DETAILS]` or `[RAW_DECOMPILED_CODE]`
   - Preserve the function signature exactly
   - Match loop structures to detected loop headers in the CFG

7. **Generate the decompiled C code.** Send the structured prompt to the LLM and request compilable C source that faithfully represents the binary's semantics.

8. **Compile and validate.** Attempt to compile the generated C with the appropriate compiler and flags (e.g., `gcc -O0 -o output.o -c output.c`). If compilation succeeds, proceed to functional testing if test cases are available.

9. **Apply compiler-in-the-loop feedback (if compilation fails).** Append the compiler's error messages to the prompt in a `[COMPILER_ERRORS]` section. Ask the LLM to fix the code while maintaining control-flow consistency with the CFG. Limit to one feedback iteration to avoid prompt bloat.

10. **Validate functional correctness.** If test inputs/outputs are available, run the compiled code against them. Compare behavior against the original binary's outputs.

## Concrete Examples

**Example 1: Decompiling a stripped x86_64 function with broken control flow**

User: "Here's Ghidra's decompiler output for a function from a stripped x86_64 binary. It won't compile. Can you fix it and produce correct C?"

Approach:
1. Parse the user's Ghidra output as the `[RAW_DECOMPILED_CODE]` section
2. If the user also provides CFG data (basic blocks, edges), encode it into the hierarchy. If not, ask the user to export CFG data from Ghidra or reconstruct approximate control flow from the pseudo-C
3. Build the structured prompt with all four sections
4. Generate C code constrained by the CFG structure
5. Attempt compilation and apply CITL feedback if needed

Structured prompt (abbreviated):
```
[TASK]
Decompile the following function into compilable C. All branches must match
the control-flow graph. Do not introduce variables not present in the source data.

[FUNCTION_CONTEXT]
Function: sub_401230
Signature: int sub_401230(int *arr, int n)
Architecture: x86_64
Basic blocks: 7
Loops: 1 (for-loop pattern)

[CFG_OVERVIEW]
Block_0 (entry) -> [Block_1]
Block_1 (loop_header) -> [Block_2, Block_6]
Block_2 -> [Block_3, Block_4]
Block_3 -> [Block_5]
Block_4 -> [Block_5]
Block_5 -> [Block_1]
Block_6 (exit) -> []

[BLOCK_DETAILS]
Block_0: STORE param_1 -> local_arr; STORE param_2 -> local_n; INT_SEXT 0 -> i
Block_1: INT_SLESS i, local_n -> cond; CBRANCH cond, Block_2; BRANCH Block_6
Block_2: LOAD local_arr[i] -> tmp; INT_SLESS tmp, 0 -> cond2; CBRANCH cond2, Block_3; BRANCH Block_4
Block_3: INT_MULT tmp, -1 -> tmp2; STORE tmp2 -> local_arr[i]
Block_4: (fall-through)
Block_5: INT_ADD i, 1 -> i; BRANCH Block_1
Block_6: RETURN 0

[RAW_DECOMPILED_CODE]
int sub_401230(int *param_1, int param_2) {
  int iVar1;
  for (iVar1 = 0; iVar1 < param_2; iVar1++) {
    if (*(param_1 + iVar1) < 0)
      *(param_1 + iVar1) = *(param_1 + iVar1) * -1;
  }
  return 0;
}
```

Output:
```c
int sub_401230(int *arr, int n) {
    for (int i = 0; i < n; i++) {
        if (arr[i] < 0) {
            arr[i] = arr[i] * -1;
        }
    }
    return 0;
}
```

**Example 2: Cross-architecture decompilation with compiler feedback**

User: "I have ARM32 firmware disassembly and Ghidra's broken pseudo-C. Help me get compilable C with correct semantics."

Approach:
1. Note the target architecture is ARM32 -- adjust expectations for calling conventions, register widths, and endianness
2. Build `[FUNCTION_CONTEXT]` noting ARM32 specifics
3. Encode CFG from user-provided Ghidra data, paying attention to ARM conditional execution patterns
4. Generate first-pass C code
5. Compile with `arm-none-eabi-gcc` (or user's cross-compiler)
6. If errors occur, feed them back in `[COMPILER_ERRORS]` section

Compiler feedback iteration:
```
[COMPILER_ERRORS]
output.c:12:5: error: implicit declaration of function 'memcpy'
output.c:18:12: error: 'uint32_t' undeclared (first use in this function)

[INSTRUCTION]
Fix these compilation errors while maintaining the control-flow structure
from [CFG_OVERVIEW]. Add necessary includes but do not alter branch logic.
```

Output after CITL:
```c
#include <string.h>
#include <stdint.h>

void firmware_copy_buffer(uint32_t *dest, const uint32_t *src, int len) {
    if (len <= 0) return;
    memcpy(dest, src, len * sizeof(uint32_t));
    dest[len - 1] |= 0x80000000u;  // Set MSB flag on last element
}
```

**Example 3: Analyzing optimized binary with complex loop structure**

User: "This O2-optimized binary has a function with nested loops that Ghidra decompiles into spaghetti goto statements. Can you reconstruct clean C?"

Approach:
1. Extract CFG -- identify loop headers via back-edges in the graph
2. In `[CFG_OVERVIEW]`, explicitly annotate: `Block_3 (outer_loop_header)`, `Block_5 (inner_loop_header)`
3. In `[BLOCK_DETAILS]`, note the induction variables and loop bounds from P-Code
4. Instruct the LLM: "Convert goto-based control flow to structured loops. Each annotated loop_header must become a for/while loop. Nesting must match the CFG dominance structure."
5. Validate that the generated nested loops match the CFG topology

## Best Practices

- **Do:** Always include the CFG adjacency list even for simple functions -- it prevents the LLM from hallucinating branches that don't exist in the binary
- **Do:** Use stable, consistent block identifiers (Block_0, Block_1, ...) across all sections so the LLM can cross-reference between CFG_OVERVIEW and BLOCK_DETAILS
- **Do:** Annotate structural roles (loop_header, branch_target, exit) in the CFG_OVERVIEW -- this is what enables the LLM to recover structured control flow from flat goto-based decompiler output
- **Do:** Limit compiler-in-the-loop to one iteration; additional rounds cause prompt bloat and diminishing returns
- **Avoid:** Feeding only the raw decompiler pseudo-C without CFG context -- this is the baseline approach that HELIOS improves upon significantly
- **Avoid:** Introducing new global variables or function calls not grounded in the block details or raw decompilation -- enforce this explicitly in the prompt instructions
- **Avoid:** Skipping the `[FUNCTION_CONTEXT]` architecture field -- calling conventions and type widths differ across x86, ARM, and MIPS and affect correctness

## Error Handling

| Problem | Cause | Fix |
|---------|-------|-----|
| Generated code won't compile | Missing includes, type mismatches | Apply CITL: append compiler errors and re-prompt with instruction to fix while preserving CFG structure |
| Control flow doesn't match binary | LLM hallucinated branches | Re-check generated code against `[CFG_OVERVIEW]` adjacency list; re-prompt highlighting the specific mismatch |
| Ghidra CFG extraction fails | Obfuscated or packed binary | Unpack/deobfuscate first; HELIOS assumes a well-formed CFG is available |
| P-Code too verbose for context window | Very large functions (100+ blocks) | Summarize BLOCK_DETAILS to only blocks involved in control-flow decisions; omit straight-line computation blocks |
| Cross-compilation errors | Wrong compiler flags or missing cross-toolchain | Ensure the correct cross-compiler is installed (e.g., `arm-none-eabi-gcc`, `mips-linux-gnu-gcc`) and flags match the binary's original build |
| Functional correctness fails despite compilation | Semantic drift in type casts or pointer arithmetic | Compare P-Code operations in BLOCK_DETAILS against generated C line-by-line; pointer scaling and sign extension are common failure points |

## Limitations

- **Requires CFG extraction tooling.** HELIOS depends on Ghidra (or equivalent) to extract control-flow graphs and P-Code. Without this infrastructure, you cannot build the hierarchical prompt -- you fall back to plain text decompilation.
- **Single-function scope.** The technique processes one function at a time. Cross-function semantics (shared globals, callback patterns, vtable dispatch) are not captured in the hierarchy.
- **No fine-tuning advantage.** HELIOS is a prompting framework, not a trained model. For highly domain-specific binaries (custom ISAs, DSP code), a fine-tuned model may outperform.
- **Context window pressure.** Large functions with many basic blocks can exhaust the context window. Functions with 100+ blocks may need summarization, which loses detail.
- **Obfuscated binaries.** The framework assumes a clean CFG. Binaries with control-flow flattening, opaque predicates, or virtualized code will produce degraded CFGs and therefore degraded results.
- **One CITL iteration.** The compiler feedback loop is capped at one round. Deeply broken output that requires multiple fix cycles is not well-served by this approach.

## Reference

**Paper:** [HELIOS: Hierarchical Graph Abstraction for Structure-Aware LLM Decompilation](https://arxiv.org/abs/2601.14598v2) -- Achamyeleh, Thomare, Al Faruque (2026). Look for Section 3 (the four-section prompt structure and hierarchical encoding) and Section 4 (compiler-in-the-loop protocol and evaluation across six architectures).