---
name: "bridging-arithmetic-gap-cognitive"
description: "Iterative Dual-Phase Financial-PoT: decouple semantic reasoning from arithmetic computation to eliminate calculation errors in financial analysis. Use when: 'calculate financial ratios from reports', 'analyze annual report numbers', 'compute ROE/ROA from statements', 'extract and calculate metrics from financial data', 'why is my financial calculation wrong', 'build a financial analysis pipeline'."
---

This skill teaches Claude to apply the **Iterative Dual-Phase Financial-PoT** framework from the paper "Bridging the Arithmetic Gap" (Zhao et al., 2026). The core technique is architectural decoupling: instead of reasoning about numbers and computing them in one pass (which causes "Arithmetic Hallucinations" and "Cognitive Collapse"), you strictly separate semantic variable extraction from symbolic computation by generating executable Python code. This approach improved accuracy from 59.7% to 67.3% on average, with up to 10x gains on high-complexity financial reasoning tasks like cross-table synthesis and multi-step ratio calculations.

## When to Use

- When the user asks to calculate financial ratios or metrics (ROE, ROA, gross margin, turnover days, etc.) from raw financial statement data
- When the user provides tabular financial data and asks quantitative questions requiring multi-step arithmetic
- When building pipelines to extract and compute indicators from annual reports, earnings data, or financial PDFs
- When the user encounters incorrect numerical results from LLM-generated financial analysis (arithmetic hallucinations)
- When computing metrics that require cross-referencing multiple tables (e.g., Income Statement + Balance Sheet)
- When the user needs to handle unit conversions, percentage calculations, or temporal alignment (current year vs. prior year) in financial contexts
- When building automated financial analysis tools that must produce deterministic, auditable results

## Key Technique

**The Problem — Cognitive Collapse:** LLMs are probabilistic token predictors, not calculators. When financial tasks exceed a complexity threshold — particularly cross-table synthesis (linking Income Statement to Balance Sheet) or multi-step calculations (computing average balances before division) — accuracy doesn't degrade linearly. It collapses. The paper measured Qwen3-235B dropping to 6.9% accuracy on cross-table tasks using direct generation, and 3.7% on turnover-days calculations. This is "Cognitive Collapse": the model's attempt to simultaneously retrieve variables from noisy documents, align them temporally, and perform arithmetic overwhelms its reasoning capacity.

**The Solution — Dual-Phase Decoupling:** The Financial-PoT framework enforces a strict two-phase architecture. **Phase 1 (Semantic Logic Formulation)** treats the LLM exclusively as a domain analyst: it reads the financial context, normalizes entities (converting "10 billion" to numeric literals), identifies the required variables, and outputs a structured Calculation Schema — a JSON-like intermediate representation of variables and their arithmetic relationships. No computation happens here. **Phase 2 (Iterative Code Generation)** translates the schema into Python code executed in a sandbox. If execution fails (ZeroDivisionError, missing values, logical inconsistencies), the error is fed back to generate corrected code, up to 3 iterations. This closed loop converts runtime failures into self-correction signals.

**Why It Works:** The decoupling dedicates 100% of the LLM's reasoning capacity to document comprehension and variable extraction — the task it excels at. Arithmetic rigor is guaranteed by the Python interpreter. The structured schema acts as a "noise filter" between messy financial documents and deterministic computation. The paper showed this is more effective than scaling model size: Qwen3-32B with PoT (48.9%) outperformed Qwen3-235B with direct generation (45.6%).

## Step-by-Step Workflow

1. **Classify the query complexity** along three dimensions before choosing a strategy:
   - **Data Source**: Is the answer in one table (intra-table), across tables (cross-table), or directly extractable? Cross-table and multi-step queries benefit most from this technique.
   - **Mapping Difficulty**: Does the query use explicit terms matching the data ("Operating Income"), implicit terms requiring formula knowledge ("Gross Margin"), or ambiguous terms requiring semantic mapping ("CAPEX" from "Cash paid for fixed assets")?
   - **Result Unit**: Percentage, ratio, days, or currency? Days and ratio calculations are highest risk for arithmetic errors.

2. **If the task is simple extraction or single-value lookup, use direct generation.** The overhead of code generation hurts accuracy on trivial tasks. Only activate the dual-phase approach when arithmetic is involved.

3. **Phase 1 — Extract variables into a Calculation Schema.** Read the financial data and produce a structured intermediate output containing:
   - Each variable name mapped to its extracted numeric value (normalized to raw numbers, not "millions" or "billions")
   - The source table/statement for each variable
   - The time period for each variable (critical for temporal alignment)
   - The formula to apply, expressed in natural language

4. **Validate the schema for completeness.** Check that all required variables have been extracted, units are consistent, and temporal periods align (e.g., "Average Equity" requires both current and prior year values).

5. **Phase 2 — Generate Python code from the schema.** Write self-contained Python that:
   - Declares all variables explicitly from the schema
   - Implements the formula using standard arithmetic operators
   - Includes type checking and prints the final result with units
   - Handles edge cases (division by zero, missing values)

6. **Execute the code in a sandbox and capture output.** Run the generated Python and collect both the result and any errors.

7. **If execution fails, feed the error back and regenerate.** Provide the original schema, the failed code, and the error traceback. Generate corrected code. Repeat up to 3 iterations.

8. **Format the final result** with appropriate precision, units, and a brief explanation of the calculation logic for auditability.

## Concrete Examples

**Example 1: Cross-table ratio (ROE) from financial statements**

User: "Calculate the Return on Equity for 2024 given this data from the Income Statement and Balance Sheet."

```
Income Statement (2024):
  Net Income: 892,450,000 RMB

Balance Sheet:
  Total Equity Attributable to Parent (2024): 5,231,000,000 RMB
  Total Equity Attributable to Parent (2023): 4,876,000,000 RMB
```

Approach:

Phase 1 — Calculation Schema:
```json
{
  "indicator": "Return on Equity (ROE)",
  "formula": "Net Income / Average Parent Equity",
  "variables": {
    "net_income": {"value": 892450000, "source": "Income Statement", "period": "2024"},
    "equity_current": {"value": 5231000000, "source": "Balance Sheet", "period": "2024"},
    "equity_prior": {"value": 4876000000, "source": "Balance Sheet", "period": "2023"}
  },
  "unit": "percentage"
}
```

Phase 2 — Generated Python:
```python
# Variables from schema
net_income = 892_450_000
equity_current = 5_231_000_000
equity_prior = 4_876_000_000

# Calculation
avg_equity = (equity_current + equity_prior) / 2
roe = net_income / avg_equity * 100

print(f"Average Parent Equity: {avg_equity:,.2f} RMB")
print(f"ROE: {roe:.2f}%")
```

Output:
```
Average Parent Equity: 5,053,500,000.00 RMB
ROE: 17.66%
```

**Example 2: Multi-step metric (Accounts Receivable Turnover Days)**

User: "What are the AR Turnover Days for 2024? Revenue is 3.2B, AR was 410M end of 2024 and 380M end of 2023."

Approach:

Phase 1 — Calculation Schema:
```json
{
  "indicator": "Accounts Receivable Turnover Days",
  "formula": "365 / (Revenue / Average AR)",
  "variables": {
    "revenue": {"value": 3200000000, "source": "Income Statement", "period": "2024"},
    "ar_current": {"value": 410000000, "source": "Balance Sheet", "period": "2024"},
    "ar_prior": {"value": 380000000, "source": "Balance Sheet", "period": "2023"}
  },
  "unit": "days"
}
```

Phase 2 — Generated Python:
```python
revenue = 3_200_000_000
ar_current = 410_000_000
ar_prior = 380_000_000

avg_ar = (ar_current + ar_prior) / 2
ar_turnover = revenue / avg_ar
ar_days = 365 / ar_turnover

print(f"Average AR: {avg_ar:,.2f} RMB")
print(f"AR Turnover Ratio: {ar_turnover:.2f}x")
print(f"AR Turnover Days: {ar_days:.2f} days")
```

Output:
```
Average AR: 395,000,000.00 RMB
AR Turnover Ratio: 8.10x
AR Turnover Days: 45.06 days
```

**Example 3: Self-correction loop on ambiguous mapping (Free Cash Flow)**

User: "Compute FCF. Operating cash flow is 1.5B. The cash flow statement shows 'Cash paid for construction of fixed assets, intangible assets, and other long-term assets' of 320M."

Phase 1 — Calculation Schema:
```json
{
  "indicator": "Free Cash Flow (FCF)",
  "formula": "Operating Cash Flow - Capital Expenditures",
  "variables": {
    "ocf": {"value": 1500000000, "source": "Cash Flow Statement", "period": "2024"},
    "capex": {"value": 320000000, "source": "Cash Flow Statement", "period": "2024",
              "note": "Mapped from 'Cash paid for construction of fixed assets...' to CAPEX"}
  },
  "unit": "currency_rmb"
}
```

Phase 2 — Generated Python (iteration 1, suppose it errors with a typo):
```python
ocf = 1_500_000_000
capex = 320_000_000
fcf = ocf - cappex  # NameError: 'cappex' is not defined
print(f"FCF: {fcf:,.2f} RMB")
```

Self-correction — Error fed back: `NameError: name 'cappex' is not defined`

Phase 2 — Corrected Python (iteration 2):
```python
ocf = 1_500_000_000
capex = 320_000_000
fcf = ocf - capex
print(f"FCF: {fcf:,.2f} RMB")
```

Output:
```
FCF: 1,180,000,000.00 RMB
```

## Best Practices

- **Do:** Always normalize magnitude words to raw numbers in Phase 1. "Revenue of 32.5 billion" must become `32_500_000_000`, not `32.5`. Magnitude misalignment is a top source of errors.
- **Do:** Include the source statement and time period for every extracted variable. Cross-table errors almost always stem from temporal misalignment (using end-of-year equity instead of average equity).
- **Do:** Use Python's `decimal` module or explicit rounding when precision matters (e.g., percentage calculations for regulatory filings). Floating-point drift can produce incorrect results at high magnitudes.
- **Do:** Route simple extraction tasks (OCF, Debt Ratio with single-table data) directly without code generation. The paper found PoT can slightly hurt accuracy on trivial lookups due to unnecessary code synthesis overhead.
- **Avoid:** Performing arithmetic inline in natural language reasoning. Never write "892,450,000 / 5,053,500,000 = 0.1766" in prose — always offload to code. This is the entire point of architectural decoupling.
- **Avoid:** Generating monolithic code that does extraction and computation together. The two phases must remain strictly separated — the schema is the contract between them.

## Error Handling

| Error | Cause | Resolution |
|-------|-------|------------|
| `ZeroDivisionError` | Denominator variable extracted as zero (e.g., prior-year equity missing) | Check schema for missing or zero values; re-examine source data |
| Wrong magnitude | "10 billion" parsed as `10` instead of `10_000_000_000` | Enforce explicit unit normalization in Phase 1; add magnitude validation |
| Temporal mismatch | Using 2024 equity instead of average of 2023+2024 | Schema must tag each variable with its period; formulas requiring averages must pull both periods |
| Ambiguous mapping failure | "CAPEX" not found because source uses "Cash paid for fixed assets..." | Phase 1 must perform semantic mapping of non-standard terms to standard financial concepts |
| Code syntax error | Generated Python has typos or invalid syntax | Self-correction loop: feed error + original schema back, regenerate (max 3 iterations) |
| Logical inconsistency | Negative current ratio, ROE > 100% in stable company | Add sanity-check assertions in generated code (e.g., `assert current_ratio > 0`) |

## Limitations

- **Minimum model capability threshold.** The paper found that for very small models (8B parameters), the overhead of code generation outweighs benefits — the model lacks sufficient semantic understanding to produce correct schemas. This technique works best with capable models.
- **Requires structured or semi-structured input.** The framework assumes financial data can be parsed into tables or text. Scanned PDFs with poor OCR quality may introduce errors before Phase 1 even begins.
- **Not a retrieval system.** This technique handles computation once variables are located. For long documents where the right table must first be found among dozens of pages, pair this with a retrieval step (RAG or targeted extraction).
- **Diminishing returns on simple tasks.** For direct extraction or single-operation calculations (Debt Ratio = Total Liabilities / Total Assets from one table), standard generation is faster and equally accurate.
- **Domain knowledge dependency.** The Calculation Schema requires knowing the correct formula. If the user asks for a non-standard or proprietary metric, the LLM must either know the formula or be given it.

## Reference

**Paper:** Zhao, B. et al. (2026). "Bridging the Arithmetic Gap: The Cognitive Complexity Benchmark and Financial-PoT for Robust Financial Reasoning." arXiv:2601.21157v1. https://arxiv.org/abs/2601.21157v1

**Key insight to look for:** Section 4 describes the two-phase architecture (Semantic Logic Formulation + Iterative Code Generation), and Tables 2-5 show the dramatic accuracy improvements stratified by complexity dimension — particularly the cross-table and turnover-days categories where the technique produces the largest gains.