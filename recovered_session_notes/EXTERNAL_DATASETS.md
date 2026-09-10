# EXTERNAL VALIDATION — dataset survey

Compiled 2026-09-09/10 while answering reviewer Major 6 (no external
validation). Everything below was checked against the actual data or the
official documentation, not from memory. Where a fact came only from
documentation it says so.

## What this paper needs from a dataset

| Requirement | Needed for |
|---|---|
| Pixel-level mass contours | Stage 1 Dice, and the mask that feeds the pooling |
| BI-RADS assessment category recorded | Novelty 2 |
| Pathology established **independently of BI-RADS** | Novelty 2 — else circular |

**No public dataset found carries all three and is reachable.** That sentence
belongs in the Limitations section; the evidence is below.

---

## VERDICT TABLE

| Dataset | Contours | BI-RADS | Independent pathology | Reachable | Use for |
|---|---|---|---|---|---|
| TOMPEI-CMMD | yes, 1,773 masks | **no** | **yes, biopsy** | yes, TCIA | Dice + classifier |
| INbreast | yes, 116 | yes | **no** (derived) | in this repo | Dice + classifier |
| BCDR-D01 | yes, 74 masses | **no** (verified) | too sparse | GitHub mirror | **Dice only** |
| Full BCDR-DM/FM | yes, 818 / 1,517 | yes (docs) | yes (docs) | **site down** | would give Novelty 2 |
| DMID | verify | yes | **unstated** | figshare | fallback only |
| VinDr-Mammo | boxes only | yes | no | PhysioNet | no |
| MIAS | circles only | no | yes | yes | no |
| EMBED / OPTIMAM / NYU | yes | yes | yes | DUA required | not in timeline |

---

## INbreast — profiled from the copy in this repo

`INbreastDataset/INbreast Release 1.0/`

- 410 DICOMs, 16-bit FFDM, 4084 x 3328, MONOCHROME2, Siemens MammoNovation
- **116 Mass ROIs across 107 images**, polygons in `AllXML/*.xml` under
  `Point_px` (Apple plist format, `Images -> ROIs -> Point_px`)
- **50 patients, 54 breasts.** `Patient ID` is "removed" in `INbreast.csv` and
  blank in the DICOM headers, but the **filename carries a patient hash**:
  `<FileName>_<patienthash>_MG_<side>_<view>_ANON.dcm`. Group on field 2.
- Mass-subset BI-RADS: 2 (22), 3 (13), 4a (7), 4b (2), 4c (12), 5 (43), 6 (8)

**THE CIRCULARITY TRAP.** INbreast has no pathology. The universal convention
is BI-RADS 1-3 benign, 4-6 malignant, which on the mass subset gives:

| BI-RADS | images | malignant |
|---|---|---|
| 2 | 22 | 0 |
| 3 | 13 | 0 |
| 4 | 21 | 21 |
| 5 | 43 | 43 |
| 6 | 8 | 8 |

The label **is** the category. BI-RADS alone scores AUC 1.000 here, so the
per-BI-RADS decision layer cannot be evaluated on INbreast at all. Report
Dice and model AUC only, and say why Novelty 2 is absent.

Other notes: 67.3 % malignant against 46.2 % in CBIS; no BI-RADS 0 or 1 in the
mass subset; 4a/4b/4c must collapse to 4; BI-RADS 6 has no fitted threshold.

---

## TOMPEI-CMMD — the strongest option

Segmentation masks added to the original CMMD collection on TCIA.
<https://www.cancerimagingarchive.net/analysis-result/tompei-cmmd/>

- **1,385 breasts with lesions, 1,363 patients, 1,773 segmentation masks**
- Masks are JSON polygons: `cgPoints`, plus `type` and `color` tags —
  mass (blue), calcification (yellow), fibroadenoma (red), distortion (green),
  focal asymmetric density (purple), lipoma (light blue)
- **Biopsy-confirmed benign/malignant** inherited from CMMD. This is what
  INbreast lacks and it makes the classifier evaluation non-circular.
- GE Senographe DS, China, 1,775 patients scanned 2012-07 to 2016-01

**Published benchmark to compare against: Dice 0.8385** —
"TOMPEI-CMMD: An adaptive radiological feature analysis using unitary U-net
for high-precision mammography segmentation", Neurocomputing 2025,
<https://www.sciencedirect.com/science/article/abs/pii/S0925231225026542>
(pull the full citation from the publisher page; authors not verified here).

**Five traps, all must be declared in the paper:**

1. **MLO views only** — the 1,127 CC views were excluded. Cross-view lesion
   aggregation collapses; one breast = one image. Report image/breast and
   patient level only.
2. **255 benign vs 1,515 malignant lesions = 85.6 % malignant.** Accuracy will
   look inflated and specificity rests on few negatives. Lead with AUC and
   state the prevalence.
3. **Fibroadenoma is tagged separately from mass** but is itself a benign mass.
   Taking only the blue tag may discard most of the benign class. Count both
   tags before choosing, then state the rule.
4. **140 mammographically undetectable lesions were excluded**, plus 25 for
   image quality. The set is enriched for visible lesions, which flatters Dice.
5. **The annotating radiologist knew the malignant/benign outcome.** The
   official page says the assessment was made "with prior knowledge of the
   final outcomes". That biases the contours. Name it before a reviewer does.

**No BI-RADS.** The radiologist recorded *findings* (mass, calcification,
focal asymmetric density, distortion), not an assessment category. Novelty 2
still not testable.

---

## BCDR — verified by opening the file, not the documentation

### BCDR-D01 as distributed at `github.com/iamtariqul/BCDR-Dataset-for-Breast-Cancer`

Contains `BCDR_D01 Dataset/Image.rar` (14 MB) and
`BCDR_D01 Dataset Original.xlsx`. Sheet "Full Dataset Outlines", 143 rows,
64 patients, 79 lesions. **Every column:**

```
patient_id, age, study_id, no_lesions, lesions, breast_density,
segmentation_id, segmentation_area,
mammography_nodule, mammography_microcalcification, mammography_calcification,
mammography_axillary_adenopathy, mammography_architectural_distortion,
mammography_stroma_distortion,
classification (lesion_characterization),
biopsy (aspiration, vacuum_core, echography)
```

- **NO BI-RADS COLUMN.** `breast_density` is ACR density 1-4 (Note sheet:
  `<25%-1, 25-50%-2, 51-75%-3, >75%-4`), which is not an assessment category.
- **`classification` is almost empty**: No 118, Malign 17, Suspect 5,
  Insufficient 2, **Benign 1**. Only 29 of 143 rows have `biopsy = 1`.
  17 malignant against 1 benign — unusable for a classifier.
- **74 rows are flagged `mammography_nodule` = mass.** That is the one useful
  thing here: a free, immediate, third-domain **segmentation-only** test.

### The rest of BCDR

- BCDR-D02: 230 lesions, 455 segmentations. Documentation states it is binary
  "due to the initial BI-RADS classification of the radiologist being replaced
  by the result of the biopsy" — so the packaged benchmark subsets **strip the
  BI-RADS out**. Do not download D01-D02/F01-F03 expecting it.
- Full BCDR-FM (1,010 patients, 1,517 segmentations, digitised film — same
  domain as CBIS, so no domain shift) and BCDR-DM (724 patients, 818
  segmentations, FFDM). Documentation says cases are BI-RADS classified *and*
  biopsy-proven. **Unverified** — it is behind registration.
- **Site unreachable.** `https://bcdr.eu/`, `https://bcdr.eu/information/about`
  and the mirror `https://bcdr.ceta-ciemat.es/information/about` all returned
  **HTTP 523** (Cloudflare: origin unreachable) on 2026-09-09/10. Re-check
  before planning any work on it.

---

## DMID — fallback only

Oza P et al., *Digital mammography dataset for breast cancer diagnosis research
(DMID) with breast mass segmentation analysis*, Biomed Eng Lett 2024;14(2):
317-330, doi 10.1007/s13534-023-00339-y (PubMed 38374902, PMC10874363).
Data at <https://figshare.com/authors/Parita_Oza/17353984>.

510 images, 225 cases, Siemens MAMMOMAT 3000 Nova, ~4743x6000. 199 normal /
311 abnormal (~165 benign, ~146 malignant, 36 "not defined"). 269 have masks.

**Two red flags.**
1. The paper never says the benign/malignant class was biopsy-confirmed. If it
   is radiologist-assigned, DMID has the same circularity as INbreast.
2. The structured metadata CSV is **MIAS-format** — reference number,
   laterality, background tissue, abnormality type, class, **x/y centre and
   approximate radius in pixels**. A circle, not a contour. A separate ROI
   Masks folder exists; open one and check it is not a rasterised circle
   before trusting any Dice from it.

No mass-specific count is published.

---

## PLAN OF RECORD

1. **INbreast** — zero download, crops cell already written (`CELL X1`).
   External Dice + classifier AUC, with the circularity caveat.
2. **BCDR-D01** — 74 mass segmentations, already downloaded. Dice only.
3. **TOMPEI-CMMD** — the one that matters. External Dice against the published
   0.8385, plus classifier AUC on biopsy-proven labels. Start the download
   early; it is multi-gigabyte.
4. **Full BCDR** — only if `bcdr.eu` comes back. Would give external Novelty 2.

Everything frozen: segmenter, classifier and thresholds. No refitting.
