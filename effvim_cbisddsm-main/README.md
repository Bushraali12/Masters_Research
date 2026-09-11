CBIS-DDSM ROI classification using CNN backbone + bidirectional Mamba encoder.

# A Hybrid Architecture for Breast Cancer Classification in Mammography

This project presents a hybrid deep learning framework for breast cancer classification using mammography images. The model combines EfficientNetV2-M for local feature extraction and Vision Mamba for efficient long-range dependency modeling.

## Overview

Early and accurate breast cancer detection is critical for improving patient outcomes. Traditional CNNs capture local features effectively but struggle with global context, while Vision Transformers are computationally expensive for high-resolution medical images.

To address this, this work combines:
- EfficientNetV2-M for hierarchical visual feature extraction
- Vision Mamba for linear-complexity global context modeling
- A lightweight and scalable hybrid pipeline for mammography classification

## Dataset

- CBIS-DDSM Dataset
- Binary Classification:
  - Benign
  - Malignant

## Features

- Hybrid CNN + State Space Model architecture
- Transfer learning with ImageNet pretrained backbone
- Vision Mamba based long-range modeling
- Weighted loss for class imbalance
- Data augmentation and reproducible preprocessing pipeline

## Model Pipeline

Input Mammogram → EfficientNetV2-M → Patch Embeddings → Vision Mamba → Global Average Pooling → Classification Head

## Results

| Metric | Score |
|---|---|
| Accuracy | 94.2% |
| AUC | 0.875 |
| Sensitivity | 0.89 |
| Specificity | 0.95 |
| F1-Score | 0.90 |

## Tech Stack

- Python
- PyTorch
- Torchvision
- OpenCV
- NumPy
- Pandas

## Research Focus

- Medical Image Analysis
- Deep Learning for Healthcare
- Vision Mamba
- Computer Vision
- Mammography Classification

Colab flow:
1) Clone repo
2) Install requirements
3) Download CBIS-DDSM via Kaggle into data/raw/cbis_ddsm
4) Prepare splits: python -m src.data.prepare_cbis_ddsm --raw_dir data/raw/cbis_ddsm --out_csv data/processed/splits.csv
5) Train: python train.py --config configs/cbis_roi.yaml
6) Eval: python eval.py --config configs/cbis_roi.yaml --ckpt outputs/best.pt
