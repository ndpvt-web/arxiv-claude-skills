---
name: "automating-computational-reproducibility-social"
description: "Diagnose and repair failing computational research code to restore reproducibility. Uses an agent-based iterative workflow: inspect files, identify failures (missing packages, broken paths, version conflicts, missing logic), apply targeted fixes, and rerun in isolated environments. Trigger phrases: 'reproduce this analysis', 'fix this R script', 'make this code reproducible', 'debug this research pipeline', 'repair computational workflow', 'rerun this study'"
---

# Automating Computational Reproducibility Repair

This skill enables Claude to systematically diagnose and repair failing computational research code -- particularly R-based social science analyses -- using an agent-based iterative repair workflow. Rather than asking the user to manually debug missing packages, broken file paths, version conflicts, or incomplete logic, Claude autonomously inspects the workspace, classifies failures by complexity, applies targeted fixes, and reruns analyses until they succeed. The approach is grounded in Shah, Hopfgartner & Bleier (2026), which demonstrated that agent-based repair achieves 69-96% success rates across all failure complexity levels, substantially outperforming prompt-based approaches (31-79%).

## When to Use

- When a user shares R scripts (or other analysis code) that fail to run and asks for help reproducing results
- When reproducing a published study and encountering package installation failures, missing dependencies, or broken file paths
- When debugging research code that uses deprecated functions or has version conflicts between packages
- When analysis code is missing logic blocks, incomplete preprocessing steps, or has multi-file dependency issues
- When setting up a Docker or containerized environment to reproduce a computational study
- When a user says "this code used to work but doesn't anymore" for any statistical or data analysis pipeline
- When batch-repairing multiple scripts from a research replication package

## Key Technique: Agent-Based Iterative Repair

The paper identifies three categories of reproducibility failures, ordered by complexity:

**Category A (Execution Errors):** Wrong file paths, missing packages, simple typos. These are mechanical -- the code structure is correct but the environment is wrong. Fix rates are highest here.

**Category B (Contextual Fixes):** Outdated package APIs, deprecated function calls, missing variables, small code gaps. These require understanding what the code *intended* to do, not just what it says. The fix often requires reading documentation or inferring from surrounding context.

**Category C (Structural Errors):** Missing functions, incomplete logic blocks, multi-file dependency chains. These are the hardest -- the code is genuinely incomplete and the repair requires reconstructing the author's analytical intent from the paper text, comments, or partial implementations.

The critical insight is that **agent-based repair dramatically outperforms prompt-based repair**, especially on Category B and C failures. Prompt-based approaches (feeding error logs to an LLM and asking for corrected code) plateau quickly because they lack workspace awareness -- they cannot explore files, check what packages are actually installed, or verify that a fix works before returning it. Agent-based workflows close this gap by operating in a loop: inspect the workspace, hypothesize a fix, apply it, rerun the code, and iterate on new errors. The paper found that providing *more context* (paper text, supporting scripts, detailed instructions) helps prompt-based approaches on complex failures but is less critical for agents, since agents can gather their own context by reading files.

## Step-by-Step Workflow

1. **Inventory the workspace.** List all code files, data files, and documentation. Identify the main analysis script(s), supporting utilities, and expected input/output files. Read any README, Makefile, or manifest that describes the intended execution order.

2. **Attempt an initial run.** Execute the main script in its intended environment (R, Python, etc.) and capture the full error output -- both stderr and stdout. Do not try to fix anything yet; the goal is a complete failure profile.

3. **Classify each failure by category.** Parse the error output and assign each failure to a complexity tier:
   - **Category A:** `Error in library(X): there is no package called 'X'`, `cannot open file 'path/to/data.csv'`, `object 'x' not found` due to typos
   - **Category B:** `could not find function "X"` (deprecated API), `Error in X(): unused argument`, version-specific behavior changes
   - **Category C:** Missing function definitions referenced but never defined, incomplete data transformation pipelines, logic that assumes intermediate files that are never created

4. **Fix Category A failures first.** Install missing packages, correct file paths (use `list.files()` or `dir()` to discover actual locations), fix obvious typos. These are mechanical and should be resolved before tackling harder issues.

5. **Fix Category B failures with context.** For deprecated functions, check the package changelog or documentation to find the replacement API. For missing variables, trace the data flow backward from the error to find where the variable should have been created. Use the paper text or comments as context for inferring intent.

6. **Fix Category C failures with structural reasoning.** For missing logic blocks, read the paper's methodology section to understand what the code should do. Reconstruct missing functions from their call signatures and the expected output format. For multi-file dependencies, trace the import/source chain and ensure all referenced files exist and are correct.

7. **Rerun after each fix batch.** Do not accumulate fixes silently. After addressing each category, rerun the script and capture new errors. Failures often cascade -- fixing a missing package may reveal a path error that was previously unreachable.

8. **Validate outputs against expected results.** If the paper or replication package includes expected outputs (tables, figures, statistics), compare the reproduced outputs against them. Flag any numerical discrepancies beyond floating-point tolerance.

9. **Document every change.** Maintain a repair log listing each failure encountered, its category, the fix applied, and whether the fix resolved the issue. This log is essential for transparency -- reproducibility repair should itself be reproducible.

10. **Isolate the environment.** When possible, run repairs inside a clean container (e.g., `rocker/r-ver:4.4.1` for R projects) with controlled resources. This ensures fixes are not dependent on the user's local environment and that the repaired code is genuinely portable.

## Concrete Examples

**Example 1: Missing packages and broken paths in an R replication**

```
User: I downloaded the replication code for a political science paper but
      the main script fails immediately. Can you fix it?

Approach:
1. Read all .R files in the workspace and the README
2. Run the main script (e.g., `Rscript analysis.R`) and capture errors
3. Error output shows:
   - Error in library(haven): there is no package called 'haven'
   - Error in library(fixest): there is no package called 'fixest'
4. Install missing packages: install.packages(c("haven", "fixest"))
5. Rerun. New error:
   - cannot open file './data/survey_2019.csv': No such file or directory
6. List actual files: find data is at ./Data/Survey_2019.csv (case mismatch)
7. Fix path: change './data/survey_2019.csv' to './Data/Survey_2019.csv'
8. Rerun. Script completes. Compare output table to paper's Table 2.

Repair log:
| # | Category | Error | Fix |
|---|----------|-------|-----|
| 1 | A | Missing package 'haven' | install.packages("haven") |
| 2 | A | Missing package 'fixest' | install.packages("fixest") |
| 3 | A | Wrong file path (case) | Corrected to ./Data/Survey_2019.csv |
```

**Example 2: Deprecated API and missing preprocessing logic**

```
User: This R script from a 2019 sociology paper throws errors about
      dplyr functions. It also seems like a data cleaning step is missing.

Approach:
1. Run script, capture errors:
   - Error in select_(.data, .dots = lazyeval::lazy_dots(...)):
     could not find function "select_"
   - Error in clean_responses(df): could not find function "clean_responses"
2. Classify: select_ is Category B (deprecated dplyr SE API), clean_responses
   is Category C (function defined nowhere in the codebase)
3. Fix Category B: Replace all select_() calls with select() using tidy
   evaluation. Replace mutate_() with mutate(), filter_() with filter(), etc.
4. Fix Category C: Read the paper's Data section. It describes: "We removed
   incomplete responses and recoded Likert items to numeric scales."
   Reconstruct clean_responses():

   clean_responses <- function(df) {
     df <- df[complete.cases(df[, c("q1","q2","q3","q4","q5")]), ]
     likert_cols <- c("q1","q2","q3","q4","q5")
     df[likert_cols] <- lapply(df[likert_cols], function(x) {
       as.numeric(factor(x, levels = c("Strongly Disagree",
         "Disagree", "Neutral", "Agree", "Strongly Agree")))
     })
     df
   }

5. Rerun. Script completes. Regression coefficients match Table 3 within
   rounding tolerance.

Repair log:
| # | Category | Error | Fix |
|---|----------|-------|-----|
| 1 | B | Deprecated select_() | Replaced with select() (x4 calls) |
| 2 | B | Deprecated mutate_() | Replaced with mutate() (x2 calls) |
| 3 | C | Missing clean_responses() | Reconstructed from paper methodology |
```

**Example 3: Containerized reproduction with multi-file dependency chain**

```
User: I need to reproduce results from a paper that has 8 R scripts that
      depend on each other. Some fail. Can you set up a Docker environment
      and fix everything?

Approach:
1. Create Dockerfile based on rocker/r-ver:4.4.1 with system dependencies
2. Read all 8 scripts to determine execution order from source() calls:
   01_load_data.R -> 02_clean.R -> 03_merge.R -> 04_model.R -> ...
3. Run sequentially in container, capturing errors at each stage
4. 01_load_data.R: Missing package 'readxl' (Cat A) -> install
5. 02_clean.R: Uses reshape2::melt but reshape2 not installed (Cat A),
   plus an ifelse() with wrong condition logic (Cat B) -> install package,
   fix condition by reading paper's inclusion criteria
6. 03_merge.R: References output file from a script not in the package
   (Cat C) -> Reconstruct the merge key from the paper's appendix
7. Continue through all 8 scripts, fixing and rerunning iteratively
8. Final validation: all 4 tables and 3 figures match the publication

Output: Dockerfile + patched scripts + repair log documenting 12 fixes
across Categories A (5), B (4), C (3)
```

## Best Practices

**Do:** Always attempt an initial run before reading the code in detail. The error output is the most efficient way to discover what is actually broken, rather than guessing from code inspection alone.

**Do:** Fix failures in order of complexity (A before B before C). Simpler fixes often unblock code paths that reveal the true nature of harder failures. Fixing a missing package might expose the real error three lines later.

**Do:** Use the paper text as a repair oracle for Category C failures. When code logic is missing, the methodology section almost always describes what should happen, even if the code does not implement it.

**Do:** Run repairs in isolated containers when possible. A fix that works because of a locally installed system library is not a real fix. Use `rocker/r-ver` for R projects or equivalent base images.

**Avoid:** Rewriting working code. If a section runs correctly, do not refactor it for style or modernize its API calls. The goal is reproduction, not improvement.

**Avoid:** Installing packages from source when binaries are available. Source compilation introduces its own failure modes (missing system headers, compiler version issues) that obscure the actual reproducibility problem.

**Avoid:** Guessing at missing logic without evidence. If you cannot reconstruct a Category C fix from the paper, comments, or variable names, flag it explicitly rather than inventing plausible-looking code that produces different results.

## Error Handling

**Package installation fails (system dependency missing):** Many R packages require system libraries (e.g., `libcurl-dev` for `httr`, `libxml2-dev` for `xml2`). Check the package's SystemRequirements field and install OS-level dependencies before retrying.

**Script hangs or runs indefinitely:** Set execution timeouts (the paper used 20-minute limits for agent runs). If a script does not terminate, check for infinite loops introduced by data-dependent conditions, or models that converge slowly on the provided data.

**Outputs differ numerically from the paper:** Small differences (< 0.01 in coefficients) are usually due to floating-point non-determinism or different BLAS implementations. Larger differences suggest a fix changed the analytical logic. Review Category C repairs first.

**Cascading failures after a fix:** A single fix can change the execution path and expose many new errors. This is normal. Re-classify the new errors and continue the repair loop. Do not revert a correct fix because it "caused" new errors.

**R version incompatibility:** Some code relies on behaviors that changed between R versions (e.g., `stringsAsFactors` default changed in R 4.0). Match the R version to the paper's reported environment when possible, or add explicit `stringsAsFactors = FALSE` calls.

## Limitations

- **Missing data:** If the replication package does not include the actual dataset (common with proprietary or restricted-access data), no amount of code repair can reproduce the results. This skill only addresses code and environment failures, not data availability.
- **Non-R languages:** The paper's testbed is R-specific. The workflow generalizes to Python, Stata, and Julia, but the specific failure patterns (CRAN package resolution, R version quirks) are R-focused.
- **Intentionally obfuscated code:** If the original code was not written to be reproducible (no comments, single-letter variables, no documentation), Category C repairs become speculative. The repair quality depends heavily on the paper's methodology description.
- **Hardware-dependent results:** GPU-dependent analyses, parallel random number generation, or platform-specific floating-point behavior may produce different results even with correct code. These are not code failures and cannot be "fixed."
- **Closed-source dependencies:** If the code depends on proprietary packages, commercial software (SAS, MATLAB), or APIs that no longer exist, automated repair cannot substitute those dependencies.

## Reference

Shah, S.M.H., Hopfgartner, F., & Bleier, A. (2026). *Automating Computational Reproducibility in Social Science: Comparing Prompt-Based and Agent-Based Approaches.* arXiv:2602.08561v1. Key finding: agent-based workflows (69-96% success) substantially outperform prompt-based approaches (31-79%) for automated reproducibility repair, with the gap widening as failure complexity increases. The paper's three-category failure taxonomy (execution errors, contextual fixes, structural errors) provides a practical classification scheme for triaging reproducibility failures.