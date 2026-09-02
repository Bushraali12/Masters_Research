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

## Two sets of headline numbers — reconcile before citing

The artifacts and the transcript report **different figures for the same
pipeline**, almost certainly because they are different evaluation protocols.
This is unresolved and matters for the thesis.

| Metric | Artifacts (build log) | Transcript (Cell 9) |
|---|---|---|
| Segmentation Dice | **0.9065** (5-model average) | **0.8998** |
| Lesion AUC | **0.9043** | **0.9022** |
| Accuracy | **85.7%** (sens floor 0.90) | **84.6%** |

The build log attributes its numbers to the **official CBIS-DDSM test split**
(n=223); Cell 9 labels its row **patient-grouped 5-fold CV**. If that is the
distinction, both are correct and each belongs with its protocol named — but it
has not been confirmed against the source data. Do not mix rows from the two.

Not in dispute, recorded in both:

- Radiomic floor **0.6807**; mask-blind DenseNet-121 control **0.8092**
- External segmentation **0.880** on INbreast
- Cohort 1,696 ROIs / 1,005 lesions / 932 breasts / 892 patients, patient-disjoint,
  audited for zero straddling at mammogram, breast, lesion and patient level
- Reference-vs-predicted mask ablation: **0.8441 → 0.8449**
- Annotation ceiling **0.792 ± 0.108** (CBIS-DDSM masks vs. radiologist, Lee et al. 2017)

## Where the real work lives

**Not in this repository.** The pipeline outputs are on the AutoDL server under
`/root/autodl-tmp/CBIS` — folds (`unified_folds_mass.csv`), predicted masks
(`predmasks_mass/`), out-of-fold probabilities (`cv_mass_v2_oof.csv`), figures
(`figures/mass_final/`). Nothing on that server was affected by the session loss.

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
- Next stage, as decided in the transcript: **calcification**. Mass is done.
