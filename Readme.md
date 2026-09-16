# Mask-Weighted Dual Pooling for Mammographic Mass Classification

Code for a two-stage pipeline for mammographic mass diagnosis on CBIS-DDSM.

## Method
- Stage 1: DS-Attn-UNet with ASPP for segmentation
- Stage 2: Two-stream DenseNet-121 with mask-weighted dual pooling (w = 1 + 2m)
- Decision layer: per-BI-RADS thresholds under a cohort-level sensitivity floor

## Results — official CBIS-DDSM test partition (378 regions / 223 lesions)
| | value |
|---|---|
| Segmentation Dice | 0.9065 |
| Lesion AUC | 0.9043 [0.858, 0.941] |
| Lesion accuracy | 84.8% [79.7, 89.3] |
| External segmentation Dice (INbreast) | 0.8062 |

## Which code produces which table
| Manuscript item | Notebook | Cell |
|---|---|---|
| Segmentation, official split | `07_OFFICIAL_SEG_AND_CLASSIFIER.ipynb` | 3 |
| Classifier, official split | `07_OFFICIAL_SEG_AND_CLASSIFIER.ipynb` | 11 |
| Tables 6, 12, 18 (operating grid) | `08_FINAL_RESULTS_AND_ABLATIONS.ipynb` | 13 |
| Table 17 (architecture ladder) | `06_WHOLE_PIPELINE_CV.ipynb` | — |
| INbreast external validation | `INBreastfile.ipynb` | I1–I5 |

## Data
CBIS-DDSM: The Cancer Imaging Archive (TCIA).
INbreast: obtained under its own use agreement and **not redistributed here**.

## Trained weights
Not included owing to size. Available from the author on request.

## License
MIT — see `LICENSE`.
