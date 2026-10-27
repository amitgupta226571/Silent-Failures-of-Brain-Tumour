# Silent-Failures-of-Brain-Tumour


# Detecting and Controlling Silent Errors of Brain Tumor MRI Classifiers Across Datasets

Code, configuration and results for the paper *"Detecting and controlling silent errors of brain tumor MRI classifiers across datasets"* (manuscript in preparation for the SPIE *Journal of Medical Imaging*).

Brain tumor MRI classifiers trained on public datasets often report near-perfect accuracy. This project asks what happens when such a model is applied to **a different public dataset**:

1. How much does performance drop once duplicates between datasets are removed?
2. How well do standard confidence scores flag the model's errors?
3. Can a review threshold keep the **silent-failure rate (SFR)**, the fraction of cases that are wrong *and* not flagged for review, below a chosen limit?

![Study design](paper/figures/fig1_pipeline.png)

---

## Main findings

- **Accuracy drops under shift.** Mean accuracy falls by 8.6 to 14.0 percentage points, from 0.84–1.00 on held-out source data to 0.76–0.89 on target data.
- **Most errors are not flagged.** Softmax-based scores are as good as or better than more complex ones, with failure AUROC 0.76–0.82. Even so, with 20% of cases reviewed, about half of the errors stay unflagged, although a perfect ranking would catch nearly all of them.
- **Source-calibrated thresholds are optimistic.** On the two binary shifts, a threshold promising SFR ≤ 1% gives 8–13% on the target.
- **Target data makes control possible, at a cost.**
  - *Reweighting with unlabeled target images* keeps the mean SFR within the 1% and 2% limits on all shifts, at 1.2–1.6× the review workload of an oracle threshold. It fails at a 5% limit.
  - *100–200 labeled target scans* bring the SFR close to the limit. It stays below the limit on two shifts and goes at most 1.3× over on the third.
- **DANN + MixStyle does not help here.** It lowers target accuracy by 1.7–7.4 points (median) and makes training 2–3× less stable across seeds than plain ERM, so **ERM is the primary configuration**.

---

## Setup

| | |
|---|---|
| **Datasets** | figshare (Cheng et al.), Br35H, SARTAJ, BTD (Nickparvar) |
| **Shifts** | Br35H → BTD (binary), Br35H → SARTAJ (binary), figshare → SARTAJ (glioma / meningioma / pituitary) |
| **Deduplication** | brain crop → pHash + dHash candidates → pixel-correlation check (≥ 0.97) → union–find groups (+ figshare patient IDs); duplicates shared between source and target are removed |
| **Splits** | source: 70% train / 10% val / 20% calibration; target: 50% pool (unlabeled or few labels) / 50% test; group-aware and stratified |
| **Backbones** | ResNet-50, DenseNet-121, EfficientNet-B0, ConvNeXt-T, Swin-T, DeiT-S, DINOv2 ViT-S/14 (timm, ImageNet / self-supervised pretrained) |
| **Seeds** | 3 per backbone and shift (21 runs per shift) |
| **Confidence scores** | MSP, entropy, MaxLogit, energy, MC-dropout (MSP, MI), kNN, learned failure heads (FH-TCP, FH-soft), 3-seed deep ensembles |
| **Risk control** | conformal risk control of SFR; calibration on source, source reweighted by density ratio (unlabeled target), 25–200 labeled target scans, or oracle |
| **Statistics** | Friedman test, Wilcoxon signed-rank with Holm correction, Mann–Whitney U, Spearman |

### Data after deduplication

| Shift | Task | Source raw | Target raw | Duplicates removed | Source train / val / cal | Target pool / test |
|---|---|---:|---:|---:|---:|---:|
| Br35H → BTD | binary | 3,000 | 1,600 | 1,081 (source) | 977 / 309 / 633 | 798 / 802 |
| Br35H → SARTAJ | binary | 3,000 | 3,264 | 459 (source) | 1,284 / 412 / 845 | 1,683 / 1,581 |
| figshare → SARTAJ | 3-class | 3,064 | 2,764 | 1,335 (target) | 1,518 / 498 / 1,043 | 711 / 699 |

> BTD is itself assembled from figshare, SARTAJ and Br35H (its no-tumor class comes from Br35H). This is why so many images had to be removed in the Br35H → BTD shift.

---

## Results

All numbers are from the primary **ERM** configuration on the **target test set**, summarized over 7 backbones × 3 seeds.

### 1. Classification

| Shift | Source accuracy | Target accuracy | Target balanced accuracy |
|---|---|---|---|
| Br35H → BTD | 0.937 – 0.987 | 0.761 – 0.892 (mean 0.835) | 0.822 – 0.905 |
| Br35H → SARTAJ | 0.988 – 0.999 | 0.816 – 0.894 (mean 0.855) | 0.700 – 0.778 |
| figshare → SARTAJ | 0.837 – 0.940 | 0.774 – 0.881 (mean 0.823) | 0.779 – 0.882 |

On BTD the models mostly **miss tumors** (sensitivity 0.68–0.88). On SARTAJ they mostly **raise false alarms on normal scans** (specificity 0.49–0.64).

### 2. Error detection (mean over 21 runs)

| Shift | Score | AURC ↓ | Failure AUROC ↑ | Errors missed @10% review | Errors missed @20% review |
|---|---|---:|---:|---:|---:|
| Br35H → BTD | MSP | 0.054 | 0.820 | 0.69 | 0.46 |
| | Ensemble MSP | 0.042 | 0.832 | 0.69 | 0.46 |
| | kNN | 0.083 | 0.687 | 0.80 | 0.64 |
| | *Best possible* | – | 1.000 | 0.36 | 0.03 |
| Br35H → SARTAJ | MSP | 0.062 | 0.758 | 0.71 | 0.52 |
| | Ensemble MSP | 0.052 | 0.761 | 0.70 | 0.52 |
| | FH-soft | 0.094 | 0.652 | 0.81 | 0.64 |
| | *Best possible* | – | 1.000 | 0.28 | 0.00 |
| figshare → SARTAJ | MSP | 0.062 | 0.814 | 0.69 | 0.46 |
| | Ensemble MSP | 0.051 | 0.821 | 0.68 | 0.46 |
| | FH-soft | 0.114 | 0.677 | 0.77 | 0.62 |
| | *Best possible* | – | 1.000 | 0.41 | 0.03 |

*"Best possible"* is the smallest fraction of errors any score could miss at that budget, given the error rate. Full table for all nine scores: [`results/tables/failure_detection_pooled_over_backbones.csv`](results/tables/failure_detection_pooled_over_backbones.csv).

![AURC heatmaps](paper/figures/fig2_aurc_heatmaps.png)

### 3. Controlling the silent-failure rate (MSP)

Realized SFR on the target test set (mean over runs) and fraction of cases flagged for review.

| Shift | Calibration | SFR @ α=1% | Flagged | SFR @ α=2% | Flagged | SFR @ α=5% | Flagged |
|---|---|---:|---:|---:|---:|---:|---:|
| Br35H → BTD | Source | **8.2%** | 0.20 | **13.4%** | 0.09 | **15.5%** | 0.03 |
| | Source, reweighted | 0.9% | 0.77 | 1.6% | 0.70 | **11.0%** | 0.16 |
| | 200 target labels | 0.6% | 0.67 | 1.4% | 0.52 | 3.9% | 0.31 |
| | Oracle | 0.9% | 0.56 | 1.9% | 0.43 | 4.9% | 0.27 |
| Br35H → SARTAJ | Source | **13.3%** | 0.03 | **13.5%** | 0.02 | **13.5%** | 0.02 |
| | Source, reweighted | 0.7% | 0.90 | 1.1% | 0.88 | **10.7%** | 0.12 |
| | 200 target labels | **1.4%** | 0.67 | **2.6%** | 0.50 | **6.0%** | 0.26 |
| | Oracle | 0.9% | 0.72 | 1.9% | 0.56 | 4.9% | 0.31 |
| figshare → SARTAJ | Source | **1.0%** | 0.66 | **3.3%** | 0.41 | **11.0%** | 0.13 |
| | Source, reweighted | 0.4% | 0.89 | 0.8% | 0.83 | 3.6% | 0.50 |
| | 200 target labels | 0.8% | 0.70 | 1.5% | 0.58 | 4.0% | 0.38 |
| | Oracle | 0.8% | 0.64 | 1.8% | 0.51 | 4.8% | 0.32 |

**Bold** = mean SFR at or above the limit (the figshare → SARTAJ source value at α = 1% is 1.04%). How often each limit was exceeded in individual runs is reported in [`results/tables/risk_control_summary.csv`](results/tables/risk_control_summary.csv).

![Risk control](paper/figures/fig3_risk_control.png)

### 4. How many labeled target scans?

With *m* labeled scans, a threshold that accepts any case exists only if 1/(*m*+1) ≤ α. So 25 scans cannot support α ≤ 2%, and 50 scans cannot support α = 1%. With 100–200 scans, the SFR is close to the limit.

![Few-label calibration](paper/figures/fig4_few_label.png)

### 5. ERM vs DANN + MixStyle

| Shift | Mean target acc. (ERM) | Mean target acc. (DANN + MixStyle) | Median paired change | Wilcoxon p |
|---|---:|---:|---:|---:|
| Br35H → BTD | 0.835 | 0.735 | −1.7 pts | 0.12 |
| Br35H → SARTAJ | 0.855 | 0.729 | −7.4 pts | 0.005 |
| figshare → SARTAJ | 0.823 | 0.684 | −6.7 pts | < 0.001 |

---

## Repository structure

```
.
├── notebooks/
│   ├── failure_detection_risk_control_kaggle.ipynb   # full pipeline: data → dedup → training → scoring → risk control → tables/figures
│   └── reaggregate_results_kaggle.ipynb              # CPU only: rebuilds all tables/figures from saved predictions
├── results/
│   ├── tables/          # primary (ERM) tables, CSV + LaTeX
│   ├── tables_dann/     # DANN + MixStyle tables
│   ├── figures/         # primary (ERM) figures
│   ├── figures_dann/
│   └── methods_summary.md   # auto-generated: datasets found, backbones, hyper-parameters used
├── paper/               # LaTeX sources, figures and refs.bib
└── README.md
```

Key result files:

| File | Content |
|---|---|
| `data_and_deduplication.csv`, `duplicates_between_datasets.csv` | data audit |
| `classification_source_heldout.csv`, `classification_target_test.csv` | accuracy, balanced accuracy, AUC, ECE, sensitivity / specificity |
| `failure_detection_target.csv`, `failure_detection_pooled_over_backbones.csv`, `failure_detection_ensembles.csv` | AURC, E-AURC, failure AUROC, FSR@budget |
| `scorer_ranks_friedman.csv`, `scorer_vs_msp_wilcoxon.csv` | statistical tests |
| `risk_control_summary.csv`, `risk_control_ensembles.csv` | SFR / FSR control for every calibration setting |
| `silent_failures_by_original_class.csv` | where unflagged errors occur |
| `dann_vs_erm_test.csv`, `dann_vs_erm_all_backbones.csv`, `ablation_*.csv` | training-method comparison and ablations |
| `shift_magnitude_*.csv` | domain-classifier AUROC vs. performance drop |

---

## How to reproduce (Kaggle)

1. **Create a Kaggle notebook** from `notebooks/failure_detection_risk_control_kaggle.ipynb`.
2. **Add the four datasets** with *+ Add Input*:
   - figshare brain tumor dataset (original `.mat` release)
   - [Br35H](https://www.kaggle.com/datasets/ahmedhamada0/brain-tumor-detection)
   - [SARTAJ](https://www.kaggle.com/datasets/sartajbhuvaji/brain-tumor-classification-mri)
   - [Brain Tumor MRI Dataset (BTD)](https://www.kaggle.com/datasets/masoudnickparvar/brain-tumor-mri-dataset)

   Datasets are found automatically by their folder structure, so no paths need to be edited.
3. **Settings → Accelerator → GPU T4.** Internet is needed for the first download of the pretrained weights.
4. **Smoke test.** Leave `RUN_MODE = "smoke"` and *Run All* (~10–20 min) to check the whole pipeline end to end.
5. **Full run.** Set `RUN_MODE = "paper"` and *Save Version → Run All*. The full study does not fit into one 12 h session. Finished runs are saved, and the notebook **resumes automatically**: add the previous version's output as an input and run again until all runs are done.
6. **Rebuild tables and figures.** Open `reaggregate_results_kaggle.ipynb` (Accelerator: None), add the finished training notebook as input, and *Run All*. It writes `paper_tables_figures.zip`.

All settings live in the configuration cell (`CFG`): datasets, shifts, backbones, seeds, splits, dedup thresholds, training and scoring parameters, risk levels, and which variant is primary (`report_variants`). Nothing else needs to be edited.

**Software used for the paper:** PyTorch 2.11, timm 1.0.29, NVIDIA Tesla T4 (Kaggle).

---

## Limitations

- **Public datasets as a proxy for new hospitals.** The shifts are between public 2D slice collections, not documented clinical sites. Patient IDs exist only for figshare, and in Br35H → BTD the no-tumor images of source and target come from the same original collection.
- **Strength of the guarantees.** The reweighted guarantee is approximate when the domains are almost perfectly separable. The few-label results draw labels from the same target dataset as the test set. SFR counts a missed tumor and a false alarm equally.

---

## Citation

If you use this code, please cite the paper (details will be added after publication):

```bibtex
@article{gupta_silent_errors_brain_mri,
  author  = {Gupta, Amit and others},
  title   = {Detecting and controlling silent errors of brain tumor {MRI} classifiers across datasets},
  journal = {Journal of Medical Imaging},
  note    = {Under review},
  year    = {2026}
}
```

Please also cite the original datasets:

- J. Cheng et al., "Enhanced performance of brain tumor classification via tumor region augmentation and partition," *PLoS ONE* 10(10):e0140381, 2015.
- S. Bhuvaji et al., "Brain Tumor Classification (MRI)," Kaggle, 2020.
- A. Hamada, "Br35H: Brain Tumor Detection 2020," Kaggle, 2020.
- M. Nickparvar, "Brain Tumor MRI Dataset," Kaggle, 2021.

## License

[Choose a license, e.g. MIT for the code.] The datasets keep their original licenses; this repository does not redistribute any images.

## Contact

Amit Gupta, Delhi Technological University (DTU), New Delhi, India
