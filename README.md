# Mask-Weighted Dual Pooling for Mammographic Mass Classification

This repository contains the code for a two-stage pipeline for mammographic mass diagnosis on the CBIS-DDSM dataset.

## Method
- Stage 1: DS-Attn-UNet with ASPP for segmentation.
- Stage 2: Two-stream DenseNet-121 with mask-weighted dual pooling.
- Decision layer: Per-BI-RADS thresholds fitted under a cohort-level sensitivity constraint.

## Results
- Segmentation Dice: 0.9065
- Lesion AUC: 0.9043 [0.858, 0.941]
- Lesion accuracy: 85.7% [80.8, 90.1]

## Data
CBIS-DDSM is available from The Cancer Imaging Archive (TCIA).

## License
MIT
