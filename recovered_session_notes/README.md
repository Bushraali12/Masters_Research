# Recovered session notes

Working record recovered from the Claude Code session **"Organize CSV files and
notebooks"** (`session_0166gYKz7AQzqvsQgMFn5pxe`, opened 2026-08-03), which
became unreachable on 2026-09-02. Recovered 2026-09-02.

Each document was published as an artifact from that session and survived the
loss of its container. Both formats of each are kept: the `.html` file is the
original published page (open it in a browser), and the `.txt` file is a
plain-text extraction for grepping, diffing, and quoting into the thesis.

| Document | Published | Covers |
|---|---|---|
| `mass-diagnosis-build-log` | 2026-09-01 | The full 15-stage build, in order — what each stage was worth, plus 11 measured dead ends |
| `two-stream-mass-pipeline` | 2026-08-29 | The pipeline end to end: the two contributions and the per-stage AUC ladder |
| `segmentation-stage-dossier` | 2026-08-14 | Stage 1 in depth — crop construction, architecture, training, ablations, novel vs. standard |

## Headline results recorded in these documents

- Lesion-level AUC **0.9043** on the official CBIS-DDSM test split (n=223), up
  from a **0.6807** radiomic floor and a **0.8092** mask-blind DenseNet-121 control
- Accuracy **85.7%** at a sensitivity floor of 0.90
- Segmentation Dice **0.9065** (5-model average); **0.880** external on INbreast
- Cohort: 1,696 ROIs / 1,005 lesions / 932 breasts / 892 patients, patient-disjoint
  splits audited for zero straddling at mammogram, breast, lesion and patient level

## What was NOT recovered

The branch `claude/organize-csv-notebooks-aoqtuc` was never pushed — it does not
exist on either `Masters-Research-` or `masters_research`. Any training scripts
or intermediate outputs that lived only on that session's container are gone.
These documents are the surviving specification of that work; the numbers above
are results as reported there, not re-verified here.

The datasets and notebooks under `Csv files folder/`, `INbreastDataset/`,
`jpeg/` and `jupyternotebooks/` were committed to `main` and were never at risk.
