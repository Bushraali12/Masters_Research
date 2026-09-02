# Recovered session notes

Working record recovered from the Claude Code session **"Organize CSV files and
notebooks"** (`session_0166gYKz7AQzqvsQgMFn5pxe`, opened 2026-08-03), which
became unreachable on 2026-09-02. Recovered 2026-09-02.

## `.` — published artifacts

Each document was published as an artifact from that session and survived the
loss of its container. Both formats of each are kept: the `.html` file is the
original published page (open it in a browser), and the `.txt` file is a
plain-text extraction for grepping, diffing, and quoting into the thesis.

| Document | Published | Covers |
|---|---|---|
| `mass-diagnosis-build-log` | 2026-09-01 | The full 15-stage build, in order — what each stage was worth, plus 11 measured dead ends |
| `two-stream-mass-pipeline` | 2026-08-29 | The pipeline end to end: the two contributions and the per-stage AUC ladder |
| `segmentation-stage-dossier` | 2026-08-14 | Stage 1 in depth — crop construction, architecture, training, ablations, novel vs. standard |

## `transcript/` — conversation, recovered by paste

Parts of the conversation itself, pasted back in and saved verbatim. These carry
what the artifacts do not: the runnable cells, the reasoning, and the decisions.

| File | Covers |
|---|---|
| `01-mass-final-cells-and-lit-comparison.md` | Cells 8 and 9 (figures; literature comparison), the argument for the supervisor, the decision to stop on mass |
| `02-operating-points-wide-context-two-stream.md` | Cell 11 (comparability-tiered table), Cell 19 (all operating points, with output), Cells 20A / 20A-FIX (wide-context crops, the black-bar bug and its fix), the two-stream result, Cell 31 (ablation) |
| `03-ensemble-provenance-and-session-death.md` | The canonical ensemble number 0.9078, the conflicting archive-file cluster, the rebuild-from-primary-artifacts cell, and the diagnosed cause of the session failure |

## Results ledger — which number belongs to which protocol

Five different headline figures appear across the recovered material. They are
**not** contradictory; they are different cohorts, protocols and pipeline
versions. Nothing may be cited without its row.

| Source | Cohort / protocol | AUC | Accuracy | Dice |
|---|---|---|---|---|
| Build log artifact | official CBIS test split, n=223, sens floor 0.90 | 0.9043 | 85.7% | 0.9065 |
| Cell 9 | patient-grouped 5-fold CV | 0.9022 | 84.6% | 0.8998 |
| Cell 11 | same, image-only (no BI-RADS thresholds) | 0.9022 | **81.8%** | 0.90 |
| Cell 19 | ALL lesions, n=1005, per-BI-RADS, sens floor 0.60 | 0.9028 | 85.0% | — |
| Two-stream | ALL lesions, after wide-context stream | 0.9037 | 86.6% | — |
| Rebuilt ensemble | seven-model, pooled — **claimed canonical** | 0.9078 | — | — |

The build log row is the **official test split**; every other row is
**patient-grouped 5-fold CV**. That reconciles them. The 0.9022 → 0.9028 → 0.9037
drift across CV rows is pipeline version, not noise in one number.

### Cell 19 — the full operating-point table

| Cohort | n | AUC | global thr | per-BI-RADS | missed cancers |
|---|---|---|---|---|---|
| ALL lesions | 1005 | 0.9028 | 80.9% / 0.799 | 85.0% / 0.879 | 93 → 56 |
| BI-RADS 4 excluded (Shia protocol) | 582 | 0.9418 | 85.9% / 0.862 | **91.4% / 0.966** | 36 → 9 |
| BI-RADS 0 and 4 excluded | 496 | 0.9372 | 85.7% / 0.880 | 91.9% / 0.972 | 30 → 7 |
| biopsy-proven only (no BWC) | 908 | 0.9115 | 81.8% / 0.799 | 85.6% / 0.879 | 93 → 56 |
| BI-RADS 4 excl. + biopsy-proven | 495 | 0.9557 | 88.1% / 0.862 | **93.1% / 0.966** | 36 → 9 |

Supervisor's target was AUC 0.92–0.93 and accuracy ≥ 90%. The BI-RADS-4-excluded
cohort clears both.

### Not in dispute

- Radiomic floor **0.6807**; mask-blind DenseNet-121 control **0.8092**
- External segmentation **0.880** on INbreast
- Cohort 1,696 ROIs / 1,005 lesions / 932 breasts / 892 patients, patient-disjoint
- Reference-vs-predicted mask ablation: **0.8441 → 0.8449**
- Annotation ceiling **0.792 ± 0.108** (Lee et al. 2017)
- Measured leakage from image-level splitting: AUC **+0.089** (0.7624 → 0.8517),
  accuracy **+7.7 pts** (71.1% → 78.8%), segmentation Dice **−0.003**
- BENIGN_WITHOUT_CALLBACK: 141 / 1696 ROIs

### Two-stream (wide-context) gain

AUC 0.9021 → 0.9037, accuracy 85.2% → 86.6%, sensitivity 0.812 → 0.857,
**missed cancers 87 → 66**, false alarms 62 → 69. Picked in all five folds
despite being weaker alone (0.8652) than v2 (0.8692) — EfficientNet-B0, stronger
alone, was never picked. Decorrelation confirmed on own data.

## Defensibility risks to settle before submission

1. **BI-RADS is used twice.** The 91.4% and 93.1% rows both *exclude* a cohort by
   the radiologist's BI-RADS **and** set thresholds per BI-RADS group. Each is
   individually disclosed and defensible; stacked, they are the most likely point
   of attack. Always pair with the full-cohort 85.0% (n=1005) and the image-only
   81.8%.
2. **Sensitivity floor is 0.60.** Cell 19 optimises accuracy subject to
   sensitivity ≥ 0.60 — permissive for a cancer task. The achieved sensitivities
   are far higher (0.879–0.972), so the floor rarely binds, but the *stated*
   constraint is what a reviewer reads. The build log quotes a 0.90 floor.
3. **"Near AUC 0.95 under image-level splitting" is an extrapolation**, not a
   measurement — the +0.089 was measured on a weaker model. Label it as inference
   or the surrounding evidence-based argument inherits its weakness.
4. **BI-RADS 4 accuracy moved +2.6 pts on only +0.0055 AUC.** Possibly
   threshold-selection luck. The second seed was queued to resolve this.
5. **Four archive files disagree about the ensemble AUC** —
   `mass_lesion_errors_clean.csv` 0.9078 (and 0.9116 on another reading),
   `_final.csv` 0.9103, `_real.csv` 0.9145 fold-averaged. 0.9078 matches the
   earlier comparison-tables document and is the working answer; 0.9090 in the
   handoff notes was transcription drift. **Do not cite any of them until the
   rebuild cell in excerpt 03 reproduces the number from primary OOF files.**
   This is the blocking item for Section 5.4 of the manuscript.

## Where the real work lives

**Not in this repository.** The pipeline outputs are on the AutoDL server under
`/root/autodl-tmp/CBIS` — folds (`unified_folds_mass.csv`), predicted masks
(`predmasks_mass/`), out-of-fold probabilities (`cv_mass_v2_oof.csv`), figures
(`figures/mass_final/`). Nothing on that server was affected by the session loss.

## Why the session died

Its environment was pinned to `Bushraali12/Masters-Research-` as its source
repository. That repo was **deleted**, so every container restart failed at
`chdir /home/user/Masters-Research-: no such file or directory` before Claude
Code could start. Every "An API error occurred" was that startup failure.

The repo has since been restored (verified: `main` at `7c9d46ad`, readable
anonymously), but restoring a repo does not restore the GitHub App's
repository-access grant, so authenticated clones can still 404. Re-granting at
https://github.com/settings/installations is the one fix worth trying — though
the session stays pinned to the wrong repo either way.

**Where work belongs now:** `Bushraali12/Masters_Research` (underscore), per the
author's own instruction. `Bushraali12/Masters_Research_` is a false start and
resolves to nothing.

## What was NOT recovered

The branch `claude/organize-csv-notebooks-aoqtuc` does not exist on either
`Masters-Research-` or `masters_research`. Per the transcript, the commit
`d9cf194` was made locally but every push returned **HTTP 403** — the Claude
GitHub App lacks write access to `Masters-Research-` (grantable at
https://github.com/settings/installations). That commit only reorganized the
repo into `Images/`, `Notebooks/` and the CSV folder; no research results
depended on it.

Anything else that lived only on that session's container is gone. Nothing here
has been re-run or re-verified — these are results as reported at the time.

The datasets and notebooks under `Csv files folder/`, `INbreastDataset/`,
`jpeg/` and `jupyternotebooks/` were committed to `main` and were never at risk.

## Open items

- Confirm the protocol behind each set of headline numbers (above).
- Verify the four `CHECK` rows in Cell 9: **Tsochatzidis 2021, Tiryaki 2023,
  Salama 2021**. Salama's headline 98.87% is on **DDSM, not CBIS-DDSM**.
- Verify every row of Cell 11's `STUDIES` against full papers, not abstracts.
  **Williams 2025's 0.9771 is a validation Dice** — replace with test Dice if lower.
- Run **Cell 31** (ablation of the two-stream gain) and the **second seed**
  (`SEEDS, EPOCHS, FREEZE = [11, 22], 20, 3`) to settle whether the BI-RADS 4
  gain is real or threshold luck.
- **Cell 20B** (wide-context training) and the updated ensemble cell were promised
  but never delivered before the session was lost — they need rewriting.
- Run the rebuild cell in excerpt 03 and settle which archive file is canonical
  before Section 5.4 of the manuscript can be finished. **This is the blocking item.**
- Locate the manuscript files — recorded as "delivered and living outside the
  repo", so they are not covered by this recovery and have no durable home yet.
- Next stage, as decided in the transcript: **calcification**. Mass is done.
