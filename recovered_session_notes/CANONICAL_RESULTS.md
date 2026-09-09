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

**THE HEADLINE MODEL IS A FIVE-MEMBER ENSEMBLE.** `CELL D v2` runs
`FOLDS = [0,1,2,3,4]`, `SEEDS = [11]`, then `oof = np.mean(parts, axis=0)`
where `parts` holds the five folds' test probability vectors. One AUC is then
computed on the averaged probability. This is probability ensembling, NOT the
mean of five AUCs — say so in the abstract, methods and comparison table, or a
reviewer will compare it against single-model literature. See the
"validation rotation" section below for the per-instance numbers.

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
| BI-RADS 3 "≈5 % malignant" | see the prevalence table below | always state unit + partition |
| BI-RADS 2 "1 malignant lesion" | 1 malignant **region** (test only); **0** lesions | unit error, not a data error |
| mask gain "+0.0705" | **+0.0166 [−0.008, +0.042]** vs GAP | +0.0715 is vs seg_aux |
| "0.0039" mask-quality | **0.0040** lesion, at most **0.0075** across units | |
| 215 × 215 | **256 × 256** | |

## UNVERIFIED — no data ever supplied

INbreast: Dice 0.880, AUC 0.8935. Cannot confirm or refute.

---

## Validation rotation — the five instances behind the headline

From the saved output of `CELL D v2` (notebook 07, cell 11). Each instance is
trained on ~1,054 of the 1,318 official training regions, validated on ~264,
and scored on the identical 378-region test partition. Seed 11 throughout.

| Instance | best val AUC | test AUC (ROI) | running ensemble |
|---|---|---|---|
| fold 0 | 0.9061 | 0.8665 | 0.8665 |
| fold 1 | 0.9123 | 0.8691 | 0.8771 |
| fold 2 | 0.9067 | 0.8611 | 0.8816 |
| fold 3 | 0.8519 | 0.8402 | 0.8750 |
| fold 4 | 0.8760 | 0.8641 | **0.8769** |

- **Mean of the five AUCs = 0.8602 ± 0.0116** (range 0.8402–0.8691).
- **Ensemble of the five probability vectors = 0.8769.** Gain **+0.0167**.
- A mean can never exceed its own maximum, so any text saying the reported
  score is "the mean of the five" is describing the wrong operation.

Lesion level: single fixed-fold instance 0.8733 vs ensemble 0.9043 (+0.031).
Per-instance lesion/breast/patient AUCs were never saved — they can be
recovered from `ckpt_official/twostream_offmask_f{0..4}.npz` on the GPU box
without retraining, but only while those checkpoints survive.

**Seed averaging is a different operation from fold averaging.** Averaging
random seeds inside one fold gives **+0.0022** (Table 13 of the manuscript);
averaging the five validation-rotation folds gives **+0.0167**. The gain comes
from training-subset diversity, not initialisation noise. These are not in
conflict, but the manuscript never distinguishes them.

## Ensemble composition differs BETWEEN ladder rungs

| Rung | Cell | Models averaged | AUC (ROI) |
|---|---|---|---|
| plain DenseNet-121 | producer cell not in repo | unknown | 0.7994 |
| + mask-weighted pooling | nb07 cell 9, `TAG=v2_offmask` | **10** (5 folds × seeds 11, 22) | 0.8480 |
| + wide stream (headline) | nb07 cell 11, `TAG=twostream_offmask` | **5** (5 folds × seed 11) | 0.8769 |

The rung with MORE ensemble members scores LOWER, so the ladder cannot be an
ensembling artefact — the comparison is biased against the final model and it
still wins. Use this as the rebuttal. But the rungs are not on a common
footing and the manuscript must say so.

This also explains the stale "averaged across two seeds" sentence in the
Introduction: two seeds is true of the mask-pooling rung, not of the headline.

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
Malignant: 46.2 % ROI overall; train 48.3 % vs test 38.9 % (gap 9.4 points)
BENIGN_WITHOUT_CALLBACK: 141

## BI-RADS prevalence — ALWAYS STATE THE UNIT AND THE PARTITION

Recomputed 2026-09-09 from `unified_folds_mass.csv`. The same category has
three different prevalences depending on unit and partition. Quoting one
without saying which is how the "≈5 %" error got into the Discussion.

| BI-RADS | Lesion, full cohort (= manuscript Table 2) | Lesion, train | Region, train | Region, test |
|---|---|---|---|---|
| 0 | 86 / 12 (14.0 %) | 68 / 10 (14.7 %) | 129 / 19 (14.7 %) | 33 / 3 (9.1 %) |
| 1 | 1 / 1 (100 %) | 0 | 1 / 1 (100 %) | 2 / 2 (100 %) |
| 2 | 57 / 0 (**0 %**) | 49 / 0 (0 %) | 77 / 0 (0 %) | 14 / **1** (7.1 %) |
| 3 | 207 / 23 (11.1 %) | 155 / 22 (14.2 %) | 279 / 45 (**16.1 %**) | 85 / 4 (4.7 %) |
| 4 | 423 / 201 (47.5 %) | 324 / 161 (49.7 %) | 533 / 279 (52.3 %) | 169 / 67 (39.6 %) |
| 5 | 231 / 225 (97.4 %) | 186 / 182 (97.8 %) | 299 / 293 (**98.0 %**) | 75 / 70 (93.3 %) |
| Total | 1,005 / 462 (46.0 %) | 782 | 1,318 (48.3 %) | 378 (38.9 %) |

The manuscript's "16 % / 98 %" is the **region level, training partition**
column — the partition the thresholds are fitted on. It is NOT lesion level.

**Manuscript Table 2 already uses maximum aggregation and is correct on all
six rows.** An earlier note in this project claimed it used first-ROI. That was
wrong: `first` gives 2/59/213/422/223 lesions, which matches nothing printed.

**The BI-RADS 2 puzzle, resolved.** Patient `P_01800`, LEFT, lesion 1: the CC
view is assessed BI-RADS 2, the MLO view of the same lesion is assessed
BI-RADS 4, pathology MALIGNANT, test split. So BI-RADS 2 holds exactly one
malignant *region* (test only) and zero malignant *lesions*, because `max`
moves that lesion into category 4. Both the Methods sentence and Table 2 are
right; only the word "lesion" in §5.0.4 is wrong.

## CANNOT BE REPRODUCED FROM ANY FILE IN `Result Csv Files/`

- "image-level splitting inflates classification AUC by 0.089" —
  `officialsplit_all_runs.csv` holds 13 runs, all patient-grouped. There is no
  image-level-split run anywhere in the repo.
- "the dataset's own annotation agrees with a radiologist at Dice 0.792" —
  would require a second independent annotation of CBIS-DDSM. No such file.
- INbreast Dice 0.880 / AUC 0.8935 — no data ever supplied.

Do not print these three in the manuscript unless the source turns up.
The only protocol claim of that kind that IS verified is the 9.4-point
class-balance gap above.
