# AGENT RULES — Non-Negotiable Constraints

These rules are absolute. The agent must follow all of them in every notebook it produces.  
Violating any rule invalidates the scientific pipeline and makes the paper unpublishable.

---

## RULE 1 — Self-Containment

Every notebook must be **100% self-contained.**

- All imports at the top of the notebook.
- All `pip install` calls in the **first code cell** with `--quiet` flag.
- **No imports from any other notebook, script, or `src/` directory.**
- **No `from utils import ...`, no `sys.path.append(...)`, no relative imports.**
- If a function is needed in two notebooks, it is **copied verbatim** into both.
- The notebook must run from top to bottom on a **fresh kernel** without any prior state.

---

## RULE 2 — Data Re-Download

Every notebook that needs data must be able to **re-download it independently.**

- Each notebook has a **config cell** at the top with a flag: `REDOWNLOAD = True / False`
- If `REDOWNLOAD = True`: download fresh from GEO or the public URL
- If `REDOWNLOAD = False`: read from the notebook's own `data/` subdirectory
- The download URL must be hardcoded and documented (GEO FTP path, direct HTTP link, etc.)
- **Never assume a file exists from a previous notebook run.**

---

## RULE 3 — No Data Leakage

This is the most critical scientific rule.

- **The test set must be isolated before ANY preprocessing that uses data statistics.**
- Correct order: `train_test_split → fit scaler/normalizer on train → transform train and test separately`
- Feature selection (Boruta, LASSO, RF, XGBoost) must be fitted **only on training data** within each CV fold.
- **Variance filtering must be computed on training fold only**, then applied to validation fold.
- If this rule is violated, all reported performance metrics are invalid and the paper cannot be published.

---

## RULE 4 — Reproducibility

- `RANDOM_SEED = 42` defined in a config cell at the top of every notebook
- Every `random_state` parameter in sklearn, xgboost, etc. must be set to `RANDOM_SEED`
- `numpy` seed: `np.random.seed(RANDOM_SEED)` in every notebook
- All model training must produce **identical results on re-run** given the same data

---

## RULE 5 — Stratified Splits Only

- All train/test splits: `sklearn.model_selection.train_test_split(..., stratify=y)`
- All cross-validation: `sklearn.model_selection.StratifiedKFold`
- **Never use `KFold` without stratification.** 
- Always verify class distribution after splits: print class counts in train, val, test

---

## RULE 6 — No Overclaiming

Notebooks must not make claims the data cannot support:

- Do not report accuracy as the primary metric when classes are imbalanced
- Do not report a single train-set AUC as "model performance"
- Always report: macro-AUC, per-class F1, confusion matrix, MCC
- If a class has fewer than 10 test samples, explicitly note this as a limitation
- Do not remove poor-performing classes to inflate metrics

---

## RULE 7 — Code Quality

- All cells must have a **markdown header cell** explaining what the cell does
- No unnamed magic numbers — use named constants (`N_ESTIMATORS = 500`, `TEST_SIZE = 0.20`)
- All figures must have: title, axis labels, legend, and be saved to `outputs/` with descriptive filename
- No `print("done")` — use informative progress messages
- Catch and handle exceptions gracefully with informative error messages

---

## RULE 8 — Output Contracts

Every notebook must save its primary outputs to its own `outputs/` subdirectory.

- File formats: `.csv` for tabular data, `.pkl` (joblib) for models, `.npy` for arrays, `.png` for figures
- Every saved file must have a **comment explaining what it contains**
- The final cell of every notebook must print a **manifest** of all files saved and their sizes

---

## RULE 9 — Session Summary

Every notebook must end with a markdown cell titled `## Session Summary` containing:

```
## Session Summary

**Notebook:** NB0X — [Name]
**Completed:** [Date]

### What Was Accomplished
- [bullet list]

### Key Numbers
- Total samples: X
- Total genes (before filtering): X
- Total genes (after filtering): X
- Class distribution: {0: X, 1: X, 2: X, 3: X}

### Outputs Saved
- `outputs/file1.csv` — description
- `outputs/file2.png` — description

### Issues / Warnings
- [any problems encountered]

### Next Notebook Requires
- [what NB0(X+1) needs as input]
```

---

## RULE 10 — Fallback Documentation

If any step fails or produces results below the acceptable threshold, the notebook must:

1. Document the failure clearly in a markdown cell
2. Try the documented fallback strategy (see PROJECT_CONTEXT.md Section 7)
3. Never silently skip a step or substitute a result without documentation
4. Flag the issue so the researcher can make an informed decision

**Acceptable performance threshold:** Macro-AUC ≥ 0.75 on held-out test set. Below this, invoke fallback.

---

## RULE 11 — No Hallucinated Biology

When writing biological interpretation in markdown cells:

- Only state things that are directly supported by the gene's known function in the NAFLD/liver disease literature
- If a gene's role in NAFLD is unknown, say so explicitly: "The role of [GENE] in NAFLD progression has not been established."
- Do not fabricate citations or pathway connections
- Cite the specific SHAP value and direction (positive = pushes toward class X) when interpreting gene importance

---

## RULE 13 — Per-Class SHAP Is Mandatory

This rule exists because per-class SHAP attribution is the **primary novel contribution** of this paper. Violation makes the paper indistinguishable from prior work.

**Prohibited:**
- Reporting only a single global SHAP summary plot and calling NB06 complete
- Computing `shap_values` and averaging across all classes before analysis
- Using `shap_values.mean(axis=0)` as the primary result

**Required:**
- `shap.TreeExplainer(model).shap_values(X)` must return a list of 4 arrays (one per class)
- Each array `shap_values[k]` has shape `(n_samples, n_genes)`
- Per-class importance: `mean(|shap_values[k]|, axis=0)` computed **separately for each k**
- Top genes reported **per class**, not collapsed across classes
- The 2×2 bar chart (one subplot per class, top 20 genes each) is **the main figure** of NB06
- The global beeswarm is **supplementary only**

**Verification check the agent must run:**
```python
assert isinstance(shap_values, list), "shap_values must be a list of arrays (one per class)"
assert len(shap_values) == 4, f"Expected 4 class arrays, got {len(shap_values)}"
assert shap_values[0].shape == (n_samples, n_genes), "Shape mismatch"
# If this assertion fails, the model type may not support multi-class SHAP natively.
# Use predict_proba-based KernelExplainer as fallback.
```

---

## RULE 12 — Dependency Management

First cell of every notebook must install all dependencies. Format:

```python
# ── DEPENDENCIES ──────────────────────────────────────────────────
# Install all required packages. This cell is safe to re-run.
import subprocess, sys

packages = [
    "GEOparse",
    "pandas",
    "numpy",
    "scikit-learn",
    "xgboost",
    "boruta",
    "shap",
    "gseapy",
    "matplotlib",
    "seaborn",
    "matplotlib-venn",
    "joblib",
]

for pkg in packages:
    subprocess.check_call([sys.executable, "-m", "pip", "install", pkg, "--quiet"])

print("All dependencies installed.")
```

Adapt the package list to what the specific notebook actually uses — do not install unnecessary packages.
