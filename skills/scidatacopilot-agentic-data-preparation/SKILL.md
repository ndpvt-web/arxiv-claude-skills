---
name: "scidatacopilot-agentic-data-preparation"
description: "Build agentic pipelines that ingest heterogeneous raw scientific data, parse research intent, and produce analysis-ready unified datasets. Use when user says 'prepare scientific data', 'build a data ingestion pipeline', 'normalize heterogeneous datasets', 'parse raw experimental data', 'integrate multi-modal scientific data', or 'make this data AI-ready'."
---

# SciDataCopilot: Agentic Scientific Data Preparation

This skill enables Claude to build end-to-end agentic data preparation pipelines for heterogeneous scientific datasets. Based on the SciDataCopilot framework, it implements a four-agent architecture (Data Access, Intent Parsing, Data Processing, Data Integration) that transforms raw experimental data — CSV, JSON, FASTA, pickle, NumPy arrays, HDF5, and domain-specific formats — into unified, analysis-ready datasets. The core insight is treating data readiness as a first-class operational primitive organized around scientific tasks rather than model-centric formatting.

## When to Use

- When the user has raw scientific data in multiple formats (e.g., protein sequences in FASTA + reaction data in CSV + metadata in JSON) and needs a unified dataset
- When building a pipeline to ingest, validate, and normalize experimental data from a lab or public repository
- When the user asks to "make data AI-ready" or prepare heterogeneous data for downstream ML models
- When integrating data across scientific domains (e.g., combining genomics, proteomics, and metabolomics data)
- When the user needs automated data quality checks, audit trails, and reproducible preparation scripts for scientific workflows
- When normalizing messy tabular data with inconsistent headers, units, temporal resolutions, or missing values across files
- When the user wants to parse a natural-language research question into structured data requirements (variables, constraints, task type)

## Key Technique

SciDataCopilot introduces the **Scientific AI-Ready data paradigm**, which shifts data preparation from ad hoc, format-driven munging to a principled, task-conditioned approach. Three principles govern it: (1) **task-conditioned organization** — the scientific question determines which data units, variables, and constraints are needed; (2) **downstream compatibility** — outputs satisfy specific model input contracts (tensor shapes, column schemas, sampling rates); (3) **cross-integration ability** — heterogeneous modalities (sequence, tabular, spatial-temporal, spectral) are aligned through explicit constraint decomposition rather than forced into a single embedding space.

The framework operates through four coordinated agents over a shared knowledge base K = {D, T, C} where D is a Data Lake of normalized dataset descriptors, T is a Tool Lake of processing tools with I/O contracts, and C is a Case Lake of historical solutions. The **Data Access Agent** recursively explores directories, detects file types, and generates atomic data unit descriptors (shapes, types, value ranges, missingness). The **Intent Parsing Agent** translates a natural-language query into structured requirements R = {Obj, Var, Con, Task}, retrieves similar past solutions from the Case Lake, and produces a validated execution plan. The **Data Processing Agent** refines the plan into runnable code, executes it in a self-repairing loop (error capture, diagnosis, code regeneration, bounded retries), and generates audit reports. The **Data Integration Agent** decomposes cross-unit constraints into tool-constraint matches and orchestrates a merge pipeline that produces the final unified dataset.

The self-repairing execution loop is the key reliability mechanism: when a processing step fails, the agent captures the full error trace, diagnoses the root cause, regenerates the code conditioned on that trace, and retries — up to a bounded number of iterations. This eliminates the fragile "run once, debug manually" pattern of traditional scientific data scripts.

## Step-by-Step Workflow

1. **Scan and catalog the raw data**: Recursively walk the data directory, identify every file by extension and content sniffing (csv, json, pkl, npy, fasta, hdf5, mat, etc.), and build an inventory mapping file paths to detected formats and sizes.

2. **Generate atomic data unit descriptors**: For each file, extract a lightweight summary — column names and dtypes for tabular data, tensor shapes for arrays, sequence counts for FASTA, field lists for JSON. Record value ranges, missingness rates, and representative samples. Store these as a structured catalog (Python dict or JSON).

3. **Parse the scientific intent**: Take the user's research question or goal and decompose it into structured requirements: the research object (e.g., "enzyme catalysis reactions"), key variables (e.g., "substrate SMILES, kcat, enzyme EC number"), constraints (e.g., "only pH 7.0-7.5, temperature 25-37C"), and task type (e.g., "regression dataset for kcat prediction").

4. **Match requirements to data units**: Cross-reference the extracted requirements against the data catalog. Identify which files contain which required variables. Flag gaps where required variables are missing or where format conversion is needed.

5. **Retrieve or generate a processing plan**: If similar past pipelines exist (user-provided examples or prior runs), adapt them via differential analysis. Otherwise, generate a step-by-step plan specifying: (a) per-unit processing steps (parsing, filtering, type casting, unit conversion, resampling), (b) the tool or library for each step, and (c) I/O contracts between steps.

6. **Validate the plan before execution**: Check requirement alignment (does the plan cover all requested variables?), coverage completeness (are any data units orphaned?), and logical correctness (are I/O types compatible between steps?). Present the plan to the user for confirmation.

7. **Generate and execute processing code**: Write Python scripts (using pandas, numpy, scipy, biopython, or domain libraries as needed) for each processing step. Execute in a self-repairing loop: run the script, capture any errors, diagnose the failure, regenerate the failing section, and retry up to 3 iterations.

8. **Integrate across data units**: Define the merge strategy — join keys, temporal alignment rules, spatial alignment, or sequence-structure linkage. Execute the integration pipeline to produce a single unified dataset (DataFrame, HDF5, or structured directory).

9. **Validate output quality**: Run automated checks — row counts vs. expected, null rates, value range assertions, schema conformance, and distributional sanity checks (e.g., compare summary statistics before and after). Generate a quality report.

10. **Produce audit artifacts**: Write a provenance log documenting every transformation applied, the input-to-output lineage, any errors encountered and how they were resolved, and the final dataset schema. This ensures reproducibility.

## Concrete Examples

**Example 1: Enzyme catalysis dataset from heterogeneous sources**

User: "I have protein sequences in FASTA files, reaction data in CSVs from BRENDA, and kinetic parameters in a JSON dump from a lab database. Build me a unified dataset for predicting kcat values."

Approach:
1. Scan the directory: find `*.fasta` (3 files, 12K sequences), `*.csv` (8 files, columns vary), `kinetics.json` (nested structure with EC numbers as keys)
2. Generate descriptors: FASTA files contain UniProt IDs + sequences; CSVs have columns [EC_number, substrate, product, kcat, km, pH, temperature] but with inconsistent naming (`kcat` vs `Kcat` vs `k_cat`); JSON nests kinetic data under EC → substrate → measurements
3. Parse intent: Obj=enzyme-catalyzed reactions, Var=[sequence, EC_number, substrate_SMILES, product_SMILES, kcat, pH, temperature], Con=[kcat > 0, valid SMILES], Task=regression
4. Plan: normalize CSV column names → parse JSON into flat records → validate SMILES with RDKit → join on EC_number → merge sequences by UniProt ID → filter by constraints → output unified CSV

```python
# Step 1: Catalog data units
import os, json
from pathlib import Path
from Bio import SeqIO
import pandas as pd

catalog = {}
data_dir = Path("./raw_data")

for f in data_dir.rglob("*"):
    if f.suffix == ".fasta":
        seqs = list(SeqIO.parse(f, "fasta"))
        catalog[str(f)] = {"format": "fasta", "count": len(seqs),
                           "sample_ids": [s.id for s in seqs[:3]]}
    elif f.suffix == ".csv":
        df = pd.read_csv(f, nrows=5)
        catalog[str(f)] = {"format": "csv", "columns": list(df.columns),
                           "dtypes": {c: str(d) for c, d in df.dtypes.items()},
                           "rows": len(pd.read_csv(f))}
    elif f.suffix == ".json":
        with open(f) as jf:
            data = json.load(jf)
        catalog[str(f)] = {"format": "json",
                           "top_keys": list(data.keys())[:5],
                           "nested_depth": 3}

# Step 2: Normalize and merge
col_map = {"Kcat": "kcat", "k_cat": "kcat", "Km": "km", "k_m": "km"}
frames = []
for f, meta in catalog.items():
    if meta["format"] == "csv":
        df = pd.read_csv(f).rename(columns=col_map)
        frames.append(df)

reactions = pd.concat(frames, ignore_index=True)
reactions = reactions[reactions["kcat"] > 0].dropna(subset=["kcat"])

# Step 3: Validate SMILES
from rdkit import Chem
reactions["valid_smiles"] = reactions["substrate"].apply(
    lambda s: Chem.MolFromSmiles(s) is not None if pd.notna(s) else False
)
reactions = reactions[reactions["valid_smiles"]].drop(columns=["valid_smiles"])

# Step 4: Join sequences
seq_map = {}
for f, meta in catalog.items():
    if meta["format"] == "fasta":
        for rec in SeqIO.parse(f, "fasta"):
            seq_map[rec.id] = str(rec.seq)

reactions["sequence"] = reactions["uniprot_id"].map(seq_map)
unified = reactions.dropna(subset=["sequence"])
unified.to_csv("enzyme_kcat_dataset.csv", index=False)
```

Output: `enzyme_kcat_dataset.csv` with columns [EC_number, uniprot_id, sequence, substrate, product, kcat, km, pH, temperature] — 45,230 valid records with audit log.

---

**Example 2: Multi-modal neuroscience EEG preprocessing**

User: "I have raw MEG/EEG recordings in .fif files and behavioral logs in CSVs. Prepare a dataset for alpha-band power analysis with proper filtering and artifact removal."

Approach:
1. Scan: find `*.fif` (12 subjects), `*_events.csv` (12 matching behavioral logs)
2. Descriptors: FIF files contain 64-channel EEG at 1000 Hz, ~10 min each; CSVs contain [timestamp, condition, response_time]
3. Intent: Obj=alpha oscillations, Var=[alpha_power, channel, condition, response_time], Con=[8-13 Hz band, artifact-free epochs], Task=condition-comparison
4. Plan: load FIF → resample to 250 Hz → bandpass filter 1-40 Hz → ICA artifact rejection → epoch around events → extract alpha power via SSD → merge with behavioral data

```python
import mne
import pandas as pd
import numpy as np

results = []
for subj_id in range(1, 13):
    # Load and preprocess
    raw = mne.io.read_raw_fif(f"sub-{subj_id:02d}_meg.fif", preload=True)
    raw.resample(250)
    raw.filter(l_freq=1.0, h_freq=40.0, method="fir", phase="zero")

    # ICA artifact removal
    ica = mne.preprocessing.ICA(n_components=20, random_state=42)
    ica.fit(raw)
    eog_indices, _ = ica.find_bads_eog(raw)
    ica.exclude = eog_indices
    raw = ica.apply(raw)

    # Epoch and extract alpha power
    events_df = pd.read_csv(f"sub-{subj_id:02d}_events.csv")
    events = np.column_stack([
        (events_df["timestamp"] * raw.info["sfreq"]).astype(int),
        np.zeros(len(events_df), dtype=int),
        events_df["condition"].astype(int)
    ])
    epochs = mne.Epochs(raw, events, tmin=-0.5, tmax=1.0, baseline=(-0.5, 0))
    psd = epochs.compute_psd(fmin=8, fmax=13, method="welch")
    alpha_power = psd.get_data().mean(axis=-1)  # avg over freq bins

    subj_df = events_df.copy()
    subj_df["subject"] = subj_id
    subj_df["alpha_power_mean"] = alpha_power.mean(axis=-1)  # avg channels
    results.append(subj_df)

unified = pd.concat(results, ignore_index=True)
unified.to_csv("alpha_power_dataset.csv", index=False)
```

Output: `alpha_power_dataset.csv` with columns [subject, timestamp, condition, response_time, alpha_power_mean] — 12 subjects, ~3600 trials, artifact-free.

---

**Example 3: Earth science temporal data harmonization**

User: "I have hourly weather station CSVs, daily ocean temperature readings, and monthly sea ice extent data. Merge them into a unified daily dataset for climate analysis."

Approach:
1. Catalog: hourly CSVs (8760 rows/year, columns [datetime, temp, humidity, wind_speed, pressure]), daily ocean CSV (365 rows, [date, sst, salinity]), monthly ice CSV (12 rows, [month, extent_km2])
2. Intent: Obj=climate variables, Var=[date, air_temp, humidity, wind_speed, pressure, sst, salinity, ice_extent], Con=[daily resolution, 2023 only], Task=multivariate time-series
3. Plan: resample hourly → daily mean, forward-fill monthly → daily, align all on common date index, validate temporal coverage

```python
import pandas as pd

# Resample hourly to daily
hourly = pd.read_csv("weather_hourly.csv", parse_dates=["datetime"])
hourly = hourly.set_index("datetime")
daily_weather = hourly.resample("D").mean()

# Load daily ocean data directly
ocean = pd.read_csv("ocean_daily.csv", parse_dates=["date"]).set_index("date")

# Expand monthly to daily via forward-fill
ice = pd.read_csv("ice_monthly.csv")
ice["date"] = pd.to_datetime(ice["month"], format="%Y-%m")
ice = ice.set_index("date").resample("D").ffill()

# Merge on date index
unified = daily_weather.join(ocean, how="outer").join(ice[["extent_km2"]], how="outer")
unified = unified.loc["2023-01-01":"2023-12-31"]

# Quality report
print(f"Date range: {unified.index.min()} to {unified.index.max()}")
print(f"Rows: {len(unified)}, Null rates:\n{unified.isnull().mean().round(4)}")
unified.to_csv("climate_daily_2023.csv")
```

Output: `climate_daily_2023.csv` — 365 rows, 8 columns, with provenance log showing resampling and fill methods applied.

## Best Practices

- **Do**: Always generate a data catalog with descriptors before writing any processing code. Understanding what you have prevents wasted iterations.
- **Do**: Decompose the user's research question into explicit structured requirements (object, variables, constraints, task type) before planning the pipeline. This prevents scope drift.
- **Do**: Implement self-repairing execution — wrap processing steps in try/except blocks, capture full tracebacks, and attempt automated fixes (up to 3 retries) before failing.
- **Do**: Produce audit artifacts for every pipeline — a provenance log mapping each input file to the transformations applied and the output rows it contributed to.
- **Avoid**: Forcing all data into a single format prematurely. Keep data in its native format during cataloging and only convert during the processing phase when the target schema is known.
- **Avoid**: Skipping validation between pipeline stages. Always assert row counts, null rates, and value ranges after each transformation step — silent data corruption is the most dangerous failure mode in scientific pipelines.

## Error Handling

- **File format detection failures**: When a file extension doesn't match its content (e.g., a `.csv` that's actually tab-separated or a `.json` that's JSONL), fall back to content sniffing — read the first 1KB and detect delimiters, brackets, or binary signatures.
- **Schema mismatches across files**: When CSVs from the same source have inconsistent column names, build a column alias map and normalize before concatenation. Log every renaming.
- **Missing required variables**: When a required variable isn't found in any data unit, report the gap explicitly to the user with suggestions — can it be derived from existing columns? Is there an alternative data source?
- **Processing step failures in self-repair loop**: After 3 failed retries, stop and present the error trace to the user with a diagnosis. Common causes: out-of-memory (suggest chunked processing), missing dependencies (suggest `pip install`), malformed data rows (suggest row-level filtering).
- **Integration key mismatches**: When join keys don't align across datasets (e.g., different date formats, ID encoding differences), detect the mismatch explicitly and apply normalization before joining.

## Limitations

- This approach assumes the user can describe their scientific intent in natural language or structured terms. Purely exploratory "I don't know what I'm looking for" scenarios need an interactive discovery phase first.
- The framework works best with structured or semi-structured data. Unstructured data like lab notebook images, handwritten annotations, or raw instrument binary blobs require domain-specific parsers that may not exist.
- Self-repairing execution handles code-level errors but cannot fix fundamental data quality issues (e.g., systematic measurement bias, mislabeled samples). Domain expert review is still essential for scientific validity.
- For very large datasets (>100GB), the full-catalog-first approach may be impractical. In those cases, use sampling-based cataloging and streaming processing rather than loading everything into memory.
- The case retrieval mechanism (adapting past solutions) requires a library of prior pipelines. On first use with a new domain, plan generation falls back to LLM reasoning without case-based guidance.

## Reference

**Paper**: [SciDataCopilot: An Agentic Data Preparation Framework for AGI-driven Scientific Discovery](https://arxiv.org/abs/2602.09132v1) — Look for the four-agent architecture (Section 3), the Scientific AI-Ready paradigm formalization (Section 2), and the self-repairing execution loop in the Data Processing Agent (Section 3.3). The neuroscience and enzyme catalysis case studies (Section 4) provide the most detailed implementation examples.