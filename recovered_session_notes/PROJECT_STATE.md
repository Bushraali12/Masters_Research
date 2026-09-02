# Project state, reconstructed from the recovered material

Assembled 2026-09-02 from the three artifacts and three transcript excerpts in
this folder. Everything here traces to a recovered source. Nothing re-verified.

## The pipeline, end to end

1. **Crops.** Window sized so the lesion occupies 23% of crop area, floor
   1.15 × longest lesion dimension. Otsu breast mask, largest connected
   component; window constrained to tissue. Runs past the edge are **slid
   inward, never padded** — 153 of 1,696 needed this; 58 still touch an edge
   because those lesions genuinely run off the mammogram. Written 512², used at
   256² for segmentation. CLAHE clip 2.0, tile 8×8.
2. **Stage 1 — DS-Attn-UNet + ASPP.** Encoder 32→64→128→256; ASPP bottleneck at
   16² (1×1, dilations 6/12/18, global pool); attention gates on every skip;
   deep supervision at four depths, weights 1.0/0.5/0.3/0.2, lower three
   discarded at inference. Loss `Σ w_d [0.3·CE + 0.7·(1 − Tversky α=0.7 β=0.3)]`
   — asymmetric so missing lesion tissue costs more than over-segmenting.
   Adam 1e-3, ReduceLROnPlateau, AMP, batch 16, ≤60 epochs, early stop 10,
   8× lockstep augmentation, 4-way rotation TTA at threshold 0.5.
3. **Mask handover.** Each fold's segmenter writes masks **only for patients it
   never trained on**. The radiologist's outline never reaches the classifier.
   Patient disjointness asserted in code before every fold.
4. **Stage 2 — two-stream DenseNet-121.** Tight stream 768, wide stream 512,
   shared weights (15.75M → 8.79M params). Mask-weighted dual pooling
   `w = 1 + 2·m`: pool once weighted, once unweighted, concatenate.
   Progressive unfreezing (3 frozen epochs, then 3e-5 backbone / 3e-4 head),
   focal loss γ=2 with class weighting, three auxiliary heads (subtlety,
   mass_shape, mass_margins) at weight 0.3, 4-way TTA.
5. **Per-BI-RADS decision layer.** One threshold per BI-RADS category, fitted
   jointly by multi-start coordinate ascent under a single cohort-level
   sensitivity floor. Frozen operating point
   `{0: 0.515, 1: 0.44, 2: 0.44, 3: 0.495, 4: 0.455, 5: 0.01}`, global 0.44,
   fitted on official training out-of-fold predictions only.

## Headline result (build log, official CBIS-DDSM split)

| Unit | n | AUC [95% CI] | Accuracy [95% CI] | Sens | Spec | Missed |
|---|---|---|---|---|---|---|
| ROI | 378 | 0.8769 [0.828–0.918] | 81.0% [76.3–85.4] | 0.878 | 0.766 | 18/147 |
| **Lesion** | **223** | **0.9043 [0.858–0.941]** | **85.7% [80.8–90.1]** | 0.885 | 0.838 | 10/87 |
| Breast | 210 | 0.9016 [0.855–0.940] | 84.8% [79.6–89.5] | 0.882 | 0.824 | 10/85 |
| Patient | 201 | 0.9041 [0.859–0.944] | 85.1% [80.1–90.0] | 0.882 | 0.828 | 10/85 |

Secondary protocol, 5-fold CV: ROI 0.8734 · lesion 0.8885 · breast 0.8869 ·
patient 0.8844 — below the official split at every unit, which is the honest
direction.

## The build ladder

0.6807 radiomics → 0.8092 mask-blind DenseNet → 0.8715 training recipe →
0.8816 two-stream → **0.9043 with mask-weighted pooling** → 85.7% with the
per-BI-RADS layer. Resolution + shared weights later gave 0.8932 no-mask
(+0.0116 at 44% fewer parameters).

## What is actually novel

- **Stage 1, Novelty 1 — a neutral result with a proof.** The dueling
  value/advantage head transferred to dense segmentation is *provably
  degenerate* for two-class output: `z₀ − z₁ = A₀ − A₁`, so V cancels and
  receives zero gradient — measured at 4.3 × 10⁻⁹, float32 noise. Confirmed
  empirically (+0.0003 against 0.0030 fold noise) and structurally, via a
  repaired `duel2` variant. Not a gain; a provable negative, which is worth
  more than an unexplained small gain.
- **Stage 1, Novelty 2 — protocol.** Leakage- and geometry-controlled
  segmentation: predicted masks patient-blind, coverage-normalised crops
  (measurably removing size dependence, Spearman +0.039), zero synthetic pixels,
  one partition shared by both stages.
- **Stage 2 — the contribution with a positive number.** Mask-weighted dual
  pooling, worth **+0.0705 AUC** over an architecturally identical mask-blind
  control (+0.0227 at lesion level against the same model without the mask).
  Gating instead of weighting costs −0.11 AUC. Mechanism: the diagnostic signal
  lives at the lesion margin, which a gate discards.
- **Novelty 2 (classification) — the per-BI-RADS layer**, +5.83 lesion accuracy
  points, bootstrap +5.32 [+2.57, +8.11]. The gain comes from the ambiguous
  strata (BI-RADS 3 +11, BI-RADS 0 +5), not the easy one (BI-RADS 5 +1).

**Explicitly not novel, do not claim:** U-Net, attention gates, ASPP, deep
supervision, Tversky loss, CLAHE.

## The strongest finding in the whole project

The seven-variant pooling ablation. Sort the mask-free variants by how well they
localise — IoU 0.373, 0.686, 0.681, 0.745 — and classification degrades
monotonically: −0.0153, −0.0264, −0.0363, −0.0549. **The better the network
learns where the lesion is, the worse it classifies it.** Learning *where* costs
capacity and competes with learning *what*; the mask variant pays neither cost
because the information is supplied for free, with zero parameters and no extra
loss term.

## Self-corrections already made (all in the author's favour, credibility-wise)

- Dice across three crop generations: 0.9000 → **0.9200** → **0.9062**. The
  0.9200 was inflated by 153 crops padded with synthetic Gaussian noise, which
  the network classified as background for free. v4 replaced it with real
  parenchyma. **0.9062 is measured on harder, honest data and is the number to
  report.** Catching and reporting one's own inflation is the single most
  defensible thing in the record.
- The annotation ceiling: CBIS-DDSM's own masks score 0.792 ± 0.108 against a
  radiologist (Lee et al. 2017), so anyone reporting 0.92–0.98 is reproducing
  the curators' level-set algorithm, not a human boundary.
- Measured leakage from image-level splitting on identical model and data:
  AUC **+0.089**, accuracy **+7.7 pts**, segmentation Dice −0.003.

## Eleven measured dead ends

Cross-architecture ensembling +0.0009 · seed averaging +0.0022 · extra
cross-view aggregation +0.0008 · density+subtlety fusion −0.003 · max-vs-mean
±0.001 · view-disagreement fusion −0.0016 · calcification pretraining −0.0176 ·
EfficientNet-B0 −0.0022 · RadImageNet init failed to converge · handcrafted
radiomics too weak (0.6807) · learned attention replacing the mask −0.0153 to
−0.0549. Six were post-hoc probes on saved predictions; six nulls in a row is
itself the finding — the information in the probabilities is exhausted.

## THE CONFLICT THAT MATTERS MOST

The build log (2026-09-01) closes with this, under "Superseded, kept for the
record":

> An earlier phase used patient-grouped 5-fold CV with a **seven-model greedy
> ensemble** at a **0.70 sensitivity floor**, reaching lesion **AUC 0.9089 and
> 87.0% accuracy**. It was retired in favour of the official-split single model
> because a single model on the published split is directly comparable to the
> literature, whereas a nested-selection ensemble on a custom protocol is not.
> **The earlier numbers should not be mixed into the current tables.**

But on 2026-09-02 — the last working day — the session was **rebuilding exactly
that seven-model ensemble** (`SEL` per fold: twostream, endtoend_fused,
twostream_calcpre) and hunting its canonical AUC (0.9078 vs 0.9089 vs 0.9103 vs
0.9116 vs 0.9145) for **Section 5.4 of the manuscript**.

So one of these is true, and only the author knows which:

- **(a)** The manuscript is built on the retired ensemble phase, in which case
  the build log's own instruction is being violated; or
- **(b)** The ensemble was un-retired after 2026-09-01, in which case the build
  log is stale and its headline 0.9043 / 85.7% is no longer the result.

Note also that the build log's own figure for that phase is **0.9089**, while
the last session concluded **0.9078** was canonical and called 0.9090
"transcription drift" — the build log may itself carry the drifted number.

**Three different sensitivity floors are in play: 0.90 (build log), 0.70
(retired ensemble phase), 0.60 (Cell 19).** Each produces different accuracies
from the same model.

**Nothing should be written into the thesis until (a) vs (b) is settled.**

## What remains, per the build log

- One optional 5-fold ensemble run over the official training set (~3.5 h,
  plausibly +0.005 to +0.015)
- Grad-CAM figure validated quantitatively against ground-truth boundaries — a
  measurement the comparator papers cannot make, because they lack the masks
- All four tables computed and saved
- Whole-mammogram extension prepared on disk (2,089 train / 369 val / 362 test),
  estimated 5–9 weeks, expected outcome at or below current numbers
- Then: calcification, where architectural distortion sits at 0.513 — chance

> The experiments are finished. Everything left is on the page, not the GPU.

## What is NOT recovered

Cell A1; Cells 1–7, 10, 12–18, 20B, 21–30; the Cell 31 subgroup output; the
manuscript files and Section 5.4; `ablation_AB.js`; the entire calcification
stage; the per-variant probability files and logged runs
(`officialsplit_all_runs.csv`, `pooling_ablation_768.csv`) that every figure
traces to. All of these live on the AutoDL server at `/root/autodl-tmp/CBIS`,
not in any recovered artifact.
