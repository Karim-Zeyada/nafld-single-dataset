# PROJECT CONTEXT — NAFLD Single-Dataset Path
> **Single source of truth for this sub-project.**  
> Read before every session. Every notebook, every decision, every output must align with what is written here.

---

## 1. Why This Sub-Project Exists

The original multi-dataset pipeline (GSE48452 + GSE126848 + GSE89632 + GSE135251) produced models that overfit and failed to generalize on held-out test sets. The root cause: combining microarray datasets from heterogeneous platforms introduces batch effects that ComBat does not fully remove, and the small per-class sample sizes after merging create an overfitting trap.

**Decision:** Simplify to a single RNA-seq dataset with no batch problem, clean labels, and the largest per-class n available in the NAFLD GEO space.

---

## 2. Dataset: GSE135251

| Property | Value |
|---|---|
| **GEO Accession** | GSE135251 |
| **Platform** | RNA-seq (Illumina) |
| **Total samples** | 206 liver biopsies + 10 controls = ~216 total |
| **Tissue** | Human liver biopsy |
| **Source paper** | Govaere et al., *Science Translational Medicine*, 2020 |
| **DOI** | [10.1126/scitranslmed.aba4448](https://doi.org/10.1126/scitranslmed.aba4448) |
| **Batch correction needed** | **No** — single platform, single study |
| **Label availability** | NAS score, fibrosis stage (F0–F4), steatosis, inflammation, ballooning |
| **GEO URL** | https://www.ncbi.nlm.nih.gov/geo/query/acc.cgi?acc=GSE135251 |
| **Download format** | Normalized count matrix (soft/series matrix or supplementary CSV) |

### 4-Class Label Mapping

| Class | Label | Mapping Rule |
|---|---|---|
| 0 | Healthy Control | Control samples with no histological disease |
| 1 | Simple Steatosis (NAFL) | NAS 1–2, Fibrosis F0, no ballooning |
| 2 | NASH | NAS ≥ 3, with lobular inflammation + ballooning, F0–F1 |
| 3 | Advanced Fibrosis | Fibrosis stage F2–F4 regardless of NAS |

> **Note on class overlap:** Some samples may qualify for Class 2 and 3 simultaneously (NASH + fibrosis). Priority rule: if fibrosis stage ≥ F2, assign Class 3. This is the clinical priority — fibrosis stage is the primary determinant of prognosis.

### Expected Class Distribution (approximate, from literature)
- Class 0: ~10–15 samples
- Class 1: ~40–50 samples  
- Class 2: ~60–70 samples
- Class 3: ~80–90 samples (F1+F2+F3+F4)

> **Class imbalance warning:** Class 0 will be small. Strategy: use stratified k-fold CV, class-weighted loss functions, and SMOTE only if needed. Do not upsample before CV split (data leakage).

---

## 3. Research Question (Simplified Scope)

> Can a multi-class machine learning model trained on a single bulk RNA-seq hepatic transcriptomic dataset (GSE135251) identify a minimal, reproducible set of stage-specific gene expression biomarkers that accurately distinguish the four stages of NAFLD progression?

---

## 4. Novel Contribution (Retained from Original Project)

All six contributions from the original project remain valid. The simplification does not dilute novelty — in fact it strengthens reproducibility claims:

1. **True 4-class staging pipeline** on a single clean RNA-seq dataset
2. **Per-class SHAP biomarker attribution** at transcriptomic scale (~20,000 genes)
3. **4-algorithm consensus extended to multi-class** (Boruta + LASSO + RF + XGBoost)
4. **Documented preprocessing** — no batch correction needed (strength, not weakness)
5. **Per-stage GO/KEGG/PPI biological validation**
6. **Fully open, reproducible Jupyter notebook pipeline**

---

## 5. Pipeline Architecture — Notebook Structure

Each notebook is **fully self-contained** and **independently reproducible**. Every NB downloads its own data, installs its own dependencies inline, and writes outputs to its own subdirectory. There are no shared utility scripts, no `src/` folder imports, no cross-notebook function calls.

```
nafld-single-dataset/
├── PROJECT_CONTEXT.md          ← This file (read first)
├── AGENT_PROMPT.md             ← Prompt for AI coding agent
├── AGENT_RULES.md              ← Hard rules the agent must follow
├── HANDOFF.md                  ← Session handoff document
│
├── NB01_data_acquisition/
│   ├── NB01_data_acquisition.ipynb
│   └── data/                   ← GSE135251 raw files land here
│
├── NB02_eda_and_labeling/
│   ├── NB02_eda_and_labeling.ipynb
│   └── data/                   ← Re-downloads or reads from NB01/data
│
├── NB03_preprocessing/
│   ├── NB03_preprocessing.ipynb
│   └── data/
│
├── NB04_feature_selection/
│   ├── NB04_feature_selection.ipynb
│   └── outputs/
│
├── NB05_model_training/
│   ├── NB05_model_training.ipynb
│   └── outputs/
│
├── NB06_shap_explainability/
│   ├── NB06_shap_explainability.ipynb
│   └── outputs/
│
└── NB07_biological_validation/
    ├── NB07_biological_validation.ipynb
    └── outputs/
```

---

## 6. Notebook Contracts

### NB01 — Data Acquisition
- Downloads GSE135251 raw/normalized expression matrix from GEO via `GEOparse` or direct URL
- Downloads associated metadata (sample characteristics, NAS scores, fibrosis stages)
- Saves: `expression_matrix_raw.csv`, `metadata.csv`
- Validates: sample count, gene count, metadata completeness

### NB02 — EDA & Labeling
- Re-downloads or reads NB01 outputs
- Applies 4-class label mapping (see Section 2)
- Produces: class distribution plot, PCA colored by class, heatmap of top variable genes
- Saves: `labeled_metadata.csv`, EDA figures
- **No modeling here — pure exploration**

### NB03 — Preprocessing
- Normalization: VST or log2(CPM+1) depending on format of downloaded matrix
- Low-variance gene filtering: remove bottom 20% by IQR
- Saves: `expression_preprocessed.csv` (samples × genes, labeled)
- Reports: number of genes before/after filtering

### NB04 — Feature Selection (4-Algorithm Consensus)
- Runs: Boruta, LASSO (cross-validated), Random Forest importance, XGBoost importance
- Produces: consensus gene ranking (intersection and union lists)
- Saves: `feature_importance_all_methods.csv`, `consensus_gene_list.csv`

### NB05 — Model Training & Evaluation
- Input: preprocessed matrix filtered to consensus genes
- Stratified 5-fold CV on all 4 classes
- Models: RF, XGBoost, SVM, Logistic Regression (softmax)
- Metrics: macro-AUC (OvR), per-class F1, confusion matrix, MCC
- Saves: best model pickle, performance tables

### NB06 — SHAP Explainability
- Input: best model from NB05
- Computes: SHAP values per sample per gene
- Produces: global beeswarm plot, per-class SHAP bar plots (top 20 genes per class)
- Saves: `shap_values.npy`, per-class top gene tables

### NB07 — Biological Validation
- Input: top SHAP genes per class from NB06
- Runs: GO enrichment (via `goatools` or `gseapy`), KEGG pathway analysis
- Produces: enrichment bar plots per stage, summary table
- Optional: STRING PPI network query via API

---

## 7. Fallback Plan

If 4-class models still underperform after simplification:

**Option A — Relabel to 3 classes:**  
Merge Class 1 (Steatosis) + Class 2 (NASH) into "Non-advanced NAFLD" vs Class 3 "Advanced Fibrosis" + keep Class 0. Yields Healthy / Active Disease / Fibrosis.

**Option B — Binary problem for conference submission:**  
Target: NASH vs Non-NASH (clinically the most actionable binary decision). This maps directly to existing high-performing literature (AUC ≥ 0.90) and gives a clean conference-quality result while the full pipeline is developed.

**Fallback journal:** IEEE BIBM, BIBM Workshop, or ACM BCB if binary — then upgrade to Frontiers in Genetics with multi-class when ready.

---

## 8. Environment & Reproducibility

- **Language:** Python 3.10+
- **No conda environment files required** — each NB installs via `pip install` in the first cell
- **No local imports** — all functions defined inline in the NB
- **Data:** downloaded fresh at the top of each NB via public API calls
- **Seed:** `RANDOM_SEED = 42` set in every NB, every stochastic call
- **Runtime target:** each NB ≤ 30 minutes on standard CPU (no GPU required)

---

## 9. Paper Status Tracking

| Section | Status |
|---|---|
| Title | ✅ Defined |
| Abstract | ⏳ Draft pending results |
| Introduction | ✅ Draft in NAFLD_Research_Paper_Template.docx |
| Related Work | ✅ Complete (25 refs in nafld_extended_literature_25refs.html) |
| Problem Formulation | ⏳ Needs formal mathematical definition |
| Methods | ⏳ Pending NB completion |
| Results | ⏳ Pending |
| Discussion | ⏳ Pending |
| Conclusion | ⏳ Pending |
| References | ✅ 23 refs in template |
