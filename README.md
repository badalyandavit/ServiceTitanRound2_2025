## README — Methodology & Statistical Toolkit

This notebook/script walks through a **three-layer analysis stack**:

| Layer                         | Goal                                                          | What we actually computed                                                                                                                                                                                                  | Test / metric used                                                                            | Why this choice                                                                                    |
| ----------------------------- | ------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------- |
| **Descriptive**               | Quantify latency & feedback by data-source mix                | *Mean* and *P99* latency for ‹PDF, Wiki, Confluence› “present vs absent”                                                                                                                                                   | Arithmetic mean, empirical 99-th percentile                                                   | Gives an immediate sense of tail behaviour vs SLA (3 500 ms).                                      |
| **Bivariate association**     | Check whether source or quality flags correlate with problems | ① φ-coefficient between **thumbs-up** and **slow**.<br>② Pearson *r* between thumbs-up and each `has_*` flag.<br>③ Correlation heat-map for latency vs presence flags.                                                     | Pearson/φ correlation                                                                         | Quick linear-association check; small sample so effect size is more useful than significance here. |
| **Contingency analysis**      | Test whether low-score buckets or PDFs affect thumbs-down     | *Two approaches depending on cell counts*<br>• **Pearson χ²** for 3 × 2 tables when expected counts ≥ 5.<br>• **Fisher exact** for 2 × 2 or sparse tables (≤ 0.85 threshold).<br>• **Cramér’s V** reported as effect size. | χ² is standard for r × c independence; Fisher ensures validity when zeros appear.             |                                                                                                    |
| **Mean-difference test**      | Validate PDF latency jump                                     | Two-sample Welch *t*-test (`ttest_ind`, unequal var)                                                                                                                                                                       | Confirms the +996 ms mean jump is statistically significant.                                  |                                                                                                    |
| **Cost / latency arithmetic** | Compare Option A vs B                                         | Token arithmetic & simple addition to the P99 tail                                                                                                                                                                         | Shows Option B breaches SLA even under best case.                                             |                                                                                                    |
| **Predictive sanity check**   | See if a toy model would flag the same drivers                | Logistic Regression, **linear SVM**, 1-hidden-layer ANN; report accuracy & ROC-AUC; dump input-space weights                                                                                                               | Sanity-check that `pdf_chunks` and `min_score` dominate; no hyper-tuning to keep runtime low. |                                                                                                    |

### Threshold choices for retrieval-score tests

| Threshold  | Rationale                                              | Test used                                                     | Result                                       |
| ---------- | ------------------------------------------------------ | ------------------------------------------------------------- | -------------------------------------------- |
| **≤ 0.80** | Strict “bad chunk” cut-off; produces very sparse table | 3 × 2 **χ²** (p = 0.0002) **and** merged-row Fisher to verify | Any chunk ≤ 0.80 → thumbs-down = 100 %.      |
| **≤ 0.85** | Mid-point sanity check                                 | 2 × 2 **Fisher** (p = 0.046)                                  | Down-vote rate triples (8.6 % → 28 %).       |
| **≤ 0.90** | More lenient; keeps counts healthy                     | 3 × 2 **χ²** (p = 0.002)                                      | Two low-score chunks drives 36 % down-votes. |

### Quick code map

```text
latency_stats()        # descriptive: mean, Δ, Pearson r, P99
bucket_significance()  # contingency: prints table, χ²/Fisher, Cramér’s V
safe_assoc_test()      # auto-falls back to Fisher when χ² assumptions fail
model_round()          # fits LogReg, linear SVM, ANN and prints metrics
```

Everything prints to console **and** saves artefacts to `./artifacts/`:

* `latency_by_source.csv` — table used for SLA and cost math
* `corr_matrix.png` — visual correlation sanity check
* `thumbs_down_low_080.png / 090.png` — bar-plots for presentations
* `latency_<source>.png` — mean latency bars for each corpus
