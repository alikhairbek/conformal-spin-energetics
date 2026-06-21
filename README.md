# conformal-spin-energetics# Conformalized Spin-State Energetics

**Calibrated, group-conditional uncertainty quantification for transition-metal spin-state splitting energies, with risk-aware spin-crossover (SCO) screening.**

This repository accompanies the manuscript *"Conformalized Spin-State Energetics: Group-Conditional Uncertainty Quantification and Risk-Aware Spin-Crossover Screening for Octahedral 3d Transition-Metal Complexes."* It provides a fully reproducible, self-contained pipeline that wraps a Gaussian-process regressor for the high-spin/low-spin splitting energy (ΔE = E(HS) − E(LS)) in **Mondrian (per metal × oxidation-state) split-conformal prediction**, yielding distribution-free, group-conditional coverage guarantees and an honest, resolution-limited triage of HS / LS / SCO candidates.

The contribution is not raw accuracy but **calibration**: turning point predictions into intervals whose coverage is guaranteed per chemical group, demonstrating where a pooled guarantee silently breaks under covariate shift, and reporting an honest resolution limit for certifying spin-crossover.

---

## Highlights

- **Group-conditional calibration.** Marginal coverage hides large per-group disparities (per-group coverage spread ≈ 0.224 at a single split; e.g. Co(III) covered at only 0.735). Mondrian conformal prediction restores near-target coverage in every group (spread ≈ 0.091; Co(III) → 0.980).
- **Sharper *and* honest.** Conformal intervals are ≈ 3.2× tighter than the native GP 90% intervals (mean width ≈ 16.4 vs ≈ 52.3 kcal/mol) while remaining calibrated.
- **Robustness under covariate shift.** Under a ligand-composition shift the pooled (marginal) guarantee drops below target (coverage 0.850), while the Mondrian guarantee holds (0.913) — the framework's main practical value.
- **Honest resolution limit.** With calibrated half-widths (~8 kcal/mol) wider than the SCO window (δ = 5 kcal/mol), **zero** complexes can be *certified* as spin-crossover by interval containment; the model is transparent about what it cannot resolve. A complementary point-prediction screen recovers SCO candidates at ~5× enrichment over the base rate.

---

## Key results (reproduced)

Base regressor (held-out test, n = 534):

| Metric | Value |
|---|---|
| MAE | 3.57 kcal/mol |
| RMSE | 4.97 kcal/mol |
| R² | 0.957 |

Coverage at the 90% target (single split, seed 0):

| Quantity | Marginal | Mondrian |
|---|---|---|
| Overall coverage | 0.901 | 0.925 |
| Per-group coverage spread | 0.224 | 0.091 |
| Co(III) coverage (worst group) | 0.735 | 0.980 |

Stability across 40 random splits (mean ± sd):

| Quantity | Marginal | Mondrian |
|---|---|---|
| Coverage | 0.893 ± 0.015 | 0.900 ± 0.018 |
| Per-group spread | 0.195 ± 0.051 | 0.148 ± 0.041 |

Out-of-distribution transfer (calibrate in-distribution, test on shifted sets):

| Test set | Marginal | Mondrian |
|---|---|---|
| In-distribution | 0.907 | 0.934 |
| Composition shift | 0.896 | 0.948 |
| Ligand shift | 0.850 | 0.913 |

Risk-aware SCO screen (point-prediction window |ΔE| < 5 kcal/mol, test set): 64 flagged, precision 0.672, recall 0.694, against a ~12% base rate (≈ 5× enrichment). Interval-certified SCO by containment: 0 (resolution limit).

> Numbers above are the exact values produced by the notebook and scripts in this repository and are deterministic for the default seed.

---

## Repository structure

```
.
├── conformal_spin/
│   ├── conformal_spin.ipynb   # self-contained notebook (no extra package needed)
│   ├── data/
│   │   ├── training_data.csv             # pooled training/calibration pool (Cr/Mn/Fe/Co × {2,3})
│   │   ├── validation_data.csv           # in-distribution validation
│   │   ├── composition_test_data.csv     # composition-shift OOD set
│   │   └── ligand_test_data.csv          # ligand-shift OOD set
│   └── README.txt                        # 3-step quick start
│
├── conformal_spin_results/
│   ├── figures/
│   │   ├── F1_coverage.png               # marginal vs Mondrian per-group coverage
│   │   ├── F2_parity.png                 # predicted vs true ΔE parity
│   │   ├── F3_triage.png                 # HS / LS / SCO triage with intervals
│   │   ├── F4_pr.png                     # precision–recall for the SCO screen
│   │   ├── F5_resolution.png             # half-width vs SCO window (resolution limit)
│   │   ├── F6_seed_robustness.png        # coverage / spread across 40 seeds
│   │   └── F7_ood_coverage.png           # coverage under covariate shift
│   ├── test_results.csv                  # per-complex predictions, intervals, group labels
│   └── sco_shortlist.csv                 # ranked spin-crossover candidates
│
├── Manuscript_Conformal_SpinState.docx   # main manuscript
├── SI_Conformal_SpinState.docx           # supporting information
├── README.md
└── LICENSE
```

If you also use the modular package version (`conformal_spin/` with `run.py` and `analysis_robustness.py`), the notebook and the package produce identical numbers; the notebook is the canonical, dependency-light entry point.

---

## Reproducing the results

### Option A — Google Colab (recommended, one click)

1. Open `conformal_spin/conformal_spin.ipynb` in Google Colab.
2. Upload the four CSV files from `conformal_spin/data/` when prompted (or mount them).
3. **Runtime → Run all.**

The notebook is self-contained — it builds the Gaussian-process model, performs Mondrian split-conformal calibration, runs the seed-robustness and covariate-shift analyses, and regenerates all seven figures (F1–F7) plus `test_results.csv` and `sco_shortlist.csv`. No custom package installation is required; it relies only on standard scientific-Python libraries (`numpy`, `pandas`, `scikit-learn`, `matplotlib`).

### Option B — Local environment

```bash
# Python 3.10+ recommended
pip install numpy pandas scikit-learn matplotlib

# then run the notebook headlessly, e.g.
jupyter nbconvert --to notebook --execute \
    conformal_spin/conformal_spin.ipynb
```

Place the four CSV files alongside the notebook (or adjust the data path at the top of the notebook). Outputs are written to a results directory mirroring `conformal_spin_results/`.

---

## Method summary

- **Target.** ΔE = E(HS) − E(LS) in kcal/mol for octahedral 3d complexes; the sign and magnitude determine the HS / LS / SCO assignment.
- **Features.** 162 leak-free descriptors — 155 graph-based revised autocorrelation (RAC) features plus 7 one-hot core (metal / oxidation-state) indicators. Geometry- and bond-length-derived features are deliberately excluded, since they collapse the uncertainty and leak information not available for screening.
- **Base model.** A `scikit-learn` `Pipeline` of `MaxAbsScaler` and a `GaussianProcessRegressor` with a composite kernel (a masked dot-product term on the core indicators plus a constant × Matérn(ν = 3/2) term over all features), `normalize_y=True`, and a fixed observation-noise level. The configuration is a faithful reconstruction of the reference model and is fully specified in the SI.
- **Conformal layer.** Split-conformal prediction with absolute-residual nonconformity scores. **Mondrian** conditioning partitions complexes by metal × oxidation state (8 groups), computing a per-group ⌈(n+1)(1−α)⌉/n quantile at α = 0.10, with a pooled fallback for very small groups.
- **Decision rules.** (i) Containment-based HS / LS / SCO triage using the calibrated interval; (ii) a point-prediction SCO screen over a tunable |ΔE| window for higher recall when certification is not required.

---

## Data sources

The dataset is assembled from published, openly available DFT spin-state data for octahedral 3d transition-metal complexes (Cr, Mn, Fe, Co in oxidation states +2 and +3). Relevant primary sources and tools include:

- Spin-state energetics dataset — *Chemical Science* (2021), DOI: [10.1039/D1SC03701C](https://doi.org/10.1039/D1SC03701C)
- Transition-metal quantum-chemistry dataset (tmQM), DOI: [10.1021/acs.jcim.0c01041](https://doi.org/10.1021/acs.jcim.0c01041)
- Associated machine-learning dataset, DOI: [10.1088/2632-2153/ad9f22](https://doi.org/10.1088/2632-2153/ad9f22) and Zenodo archive [10.5281/zenodo.13331586](https://doi.org/10.5281/zenodo.13331586)
- `molSimplify` (RAC feature generation): https://github.com/hjkgrp/molSimplify

Please consult the manuscript and SI for the full provenance, filtering, and partitioning details, and verify all bibliographic identifiers in the reference list before citing.

---

## Citation

If you use this code or the results, please cite the manuscript:

> Khairbek, A. A. *Conformalized Spin-State Energetics: Group-Conditional Uncertainty Quantification and Risk-Aware Spin-Crossover Screening for Octahedral 3d Transition-Metal Complexes.* (2026).

A `CITATION.cff` / BibTeX entry will be added upon publication; please update the year, journal, volume, and DOI to match the published version.

---

## License

Released under the MIT License. See [`LICENSE`](LICENSE).

Copyright © 2026 Ali A. Khairbek.

---

## Contact

**Ali A. Khairbek** — ORCID: [0000-0002-0477-5896](https://orcid.org/0000-0002-0477-5896)

Issues and questions are welcome via the repository's issue tracker.
