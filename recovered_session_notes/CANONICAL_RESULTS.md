# CANONICAL RESULTS — read this before writing any number

Every figure below was recomputed from the CSVs in `Result Csv Files/`
on 2026-09-09. Anything not listed here is not verified.

## THE PAIRING RULE

There are two protocols. Each has its own Dice AND its own classification
numbers. **Never take the Dice from one and the AUC from the other.**

---

## A. OFFICIAL SPLIT  ← the headline, matches manuscript Table 6

Source: `cv_mass_twostream_officialsplit_oof.csv` + `unified_folds_mass.csv`
Partition: official CBIS-DDSM, 1,318 train / 378 test regions, 0 shared patients.
Operating point: per-BI-RADS thresholds, 0.90 cohort sensitivity constraint.

**Segmentation Dice = 0.9065**  (`oof_dice_official`, 378 test regions)

| Unit | n | AUC [95% CI] | Accuracy [95% CI] | Sens | Spec | Missed |
|---|---|---|---|---|---|---|
| ROI | 378 | 0.8769 [0.828, 0.918] | 81.0 [76.3, 85.4] | 0.878 | 0.766 | 18 / 147 |
| **Lesion** | **223** | **0.9043 [0.858, 0.941]** | **85.7 [80.8, 90.1]** | **0.885** | **0.838** | **10 / 87** |
| Breast | 210 | 0.9016 [0.855, 0.940] | 84.8 [79.6, 89.5] | 0.882 | 0.824 | 10 / 85 |
| Patient | 201 | 0.9041 [0.859, 0.944] | 85.1 [80.1, 90.0] | 0.882 | 0.828 | 10 / 85 |

Single global threshold, same data: 75.66 / 79.82 / 78.57 / 79.10 %
(global thresholds: 0.44 ROI, 0.455 lesion)

Frozen per-BI-RADS thresholds, ROI level:
`{0: 0.515, 1: 0.44, 2: 0.44, 3: 0.495, 4: 0.455, 5: 0.01}`, global 0.44
Lesion level: `{0: 0.52, 2: 0.697, 3: 0.575, 4: 0.465, 5: 0.01}`, global 0.455

---

## B. PATIENT-GROUPED 5-FOLD CV  ← secondary

**Segmentation Dice = 0.8998 ≈ 0.900**  (`oof_dice`, all 1,696 regions)

Single architecture (`cv_mass_twostream_oof.csv`):
ROI 0.8734 · lesion 0.8885 · breast 0.8869 · patient 0.8844

Seven-model greedy ensemble (`mass_lesion_errors_clean.csv`, n=1,005 lesions):
- pooled lesion AUC **0.9078**
- fold-averaged **0.9117** (sd 0.0215)
- accuracy **85.97 %** (864/1005) · TP 392 FP 71 TN 472 FN 70 · sens 0.848 · spec 0.869
- best accuracy from ANY single global threshold on these preds: **84.08 %**

---

## NUMBERS THAT ARE WRONG — do not use

| Wrong | Correct | Note |
|---|---|---|
| AUC 0.9089 | **0.9078** pooled | build-log transcription drift |
| accuracy 87 % | **86.0 %** | build-log drift |
| Dice 0.9062 | 0.9065 (official) or 0.8998 (CV) | |
| BI-RADS 3 "≈5 % malignant" | 11.1 % full cohort / 16.1 % train / 4.7 % official test ROI | state which cohort |
| BI-RADS 2 "1 malignant lesion" | 0 malignant in the training partition | |
| mask gain "+0.0705" | **+0.0166 [−0.008, +0.042]** vs GAP | +0.0715 is vs seg_aux |
| "0.0039" mask-quality | **0.0040** lesion, at most **0.0075** across units | |
| 215 × 215 | **256 × 256** | |

## UNVERIFIED — no data ever supplied

INbreast: Dice 0.880, AUC 0.8935. Cannot confirm or refute.

---

## Architecture ladder, official test (AUC)

| Model | ROI | Lesion | Breast | Patient |
|---|---|---|---|---|
| DenseNet-121, tight crop | 0.7994 | 0.8092 | 0.7970 | 0.7960 |
| + mask-weighted dual pooling | 0.8480 | 0.8715 | 0.8647 | 0.8651 |
| + wide-context stream (final) | 0.8769 | 0.9043 | 0.9016 | 0.9041 |
| **Total gain** | +0.0775 | +0.0951 | +0.1046 | +0.1081 |
| Control: no mask weighting | 0.8571 | 0.8816 | 0.8746 | 0.8797 |

## Pooling ablation (768/512 shared backbone, CV masks) — lesion AUC

mask 0.9016 · gap 0.8850 · gem 0.8767 · attn 0.8697 (IoU 0.373) ·
attn_kd 0.8586 (0.686) · attn_soft 0.8486 (0.681) · seg_aux 0.8301 (0.745)
mask − gap = +0.0166 [−0.008, +0.042]. Spearman ρ across the four learned
variants = **−0.80** (not monotone: attn_soft and attn_kd invert).

## Decision layer

Gain: +5.29 ROI · +5.83 lesion · +6.19 breast · +5.97 patient
Bootstrap: +5.86 [+2.74, +9.21] lesion
Decomposition of the 20 net correct calls at ROI: BI-RADS 3 +11, 0 +5, 4 +3, 5 +1
Floor sweep, lesion: 0.70→+0.00 · 0.85→+3.14 · 0.90→+5.83 · 0.95→+8.52
Patient level is NOT monotone (dips at 0.85).

## Model without BI-RADS anywhere

Model AUC within BI-RADS 4 (category constant, 99 lesions, 40 malignant): **0.7886**
BI-RADS used alone as a predictor, lesion level: 0.8508 (model: 0.9043)

## Parameters

17,582,867 training → 16,007,426 inference. Pooling module: 0 learnable params.
Descriptor 4096 = 2 streams × 2 poolings × 1024.

## Cohort

1,696 regions · 1,005 lesions · 932 breasts · 892 patients
Official: 1,318 / 378 regions; test = 223 lesions, 210 breasts, 201 patients
Malignant: 46.2 % ROI overall; train 48.3 % vs test 38.9 %
BENIGN_WITHOUT_CALLBACK: 141
Table 2 must use **maximum** BI-RADS aggregation (matches Methods), not first-ROI.
