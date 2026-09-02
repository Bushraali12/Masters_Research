# Transcript excerpt 1 — mass stage: final figure cells + literature comparison

Recovered by paste from session `session_0166gYKz7AQzqvsQgMFn5pxe`
("Organize CSV files and notebooks"), which became unreachable 2026-09-02.
This is the tail of that conversation: the two notebook cells that close out the
mass stage, the argument to take to the supervisor, and the decision to move on
to calcification.

Verbatim as pasted. Nothing here has been re-run or re-verified.

---

## Context given with the paste

> I searched the literature to ground this in verified papers rather than
> relying only on your notebook's list. Here are both cells.

---

## CELL 8 — Visualise segmentation + classification

No GPU needed — it reads the masks and predictions already saved.

```python
# ══════════════════════════════════════════════════════════════════════
# CELL 8 — MASS: visualise segmentation AND classification   no GPU, ~1 min
#   Fig 1  segmentation: best / median / worst cases
#   Fig 2  classification: TP / TN / FP / FN examples
#   Fig 3  Dice distribution + ROC curve + confusion matrix
# ══════════════════════════════════════════════════════════════════════
import os, numpy as np, pandas as pd, cv2
import matplotlib; matplotlib.use("Agg")
import matplotlib.pyplot as plt
from sklearn.metrics import roc_curve, roc_auc_score, confusion_matrix

D   = "/root/autodl-tmp/CBIS"
FIG = os.path.join(D, "figures", "mass_final"); os.makedirs(FIG, exist_ok=True)
S   = 512
_clahe = cv2.createCLAHE(clipLimit=2.0, tileGridSize=(8, 8))

d = pd.read_csv(os.path.join(D, "unified_folds_mass.csv"))
d["assessment"] = pd.to_numeric(d["assessment"], errors="coerce")
PM = os.path.join(D, "predmasks_mass")
d["pred"] = d["img"].apply(
    lambda p: os.path.join(PM, os.path.basename(str(p)).replace("_img.png", "") + "_pred.png"))
d = d[d["pred"].apply(os.path.exists)].reset_index(drop=True)

# classification probabilities from the retrained model
v2 = os.path.join(D, "cv_mass_v2_oof.csv")
if os.path.exists(v2):
    pv = pd.read_csv(v2)[["img", "prob"]]
    pv["img"] = pv["img"].astype(str)
    d["img"] = d["img"].astype(str)
    d = d.merge(pv, on="img", how="left")
else:
    print("cv_mass_v2_oof.csv not found — classification panels will be skipped")
    d["prob"] = np.nan

print("lesions with masks: %d | with probabilities: %d" % (len(d), d["prob"].notna().sum()))


def panel(img_p, gt_p, pr_p):
    """returns (clahe image, gt mask, pred mask, colour overlay)"""
    im = cv2.imread(str(img_p), cv2.IMREAD_GRAYSCALE)
    if im is None:
        return None
    im = cv2.resize(im, (S, S)); g = _clahe.apply(im)
    gt = cv2.imread(str(gt_p), cv2.IMREAD_GRAYSCALE)
    gt = (np.zeros((S, S), np.uint8) if gt is None
          else (cv2.resize(gt, (S, S), interpolation=cv2.INTER_NEAREST) > 127).astype(np.uint8))
    pr = cv2.imread(str(pr_p), cv2.IMREAD_GRAYSCALE)
    pr = (np.zeros((S, S), np.uint8) if pr is None
          else (cv2.resize(pr, (S, S), interpolation=cv2.INTER_NEAREST) > 127).astype(np.uint8))
    ov = cv2.cvtColor(g, cv2.COLOR_GRAY2RGB)
    c, _ = cv2.findContours(gt, cv2.RETR_EXTERNAL, cv2.CHAIN_APPROX_SIMPLE)
    cv2.drawContours(ov, c, -1, (0, 255, 0), 3)          # green = truth
    c, _ = cv2.findContours(pr, cv2.RETR_EXTERNAL, cv2.CHAIN_APPROX_SIMPLE)
    cv2.drawContours(ov, c, -1, (255, 60, 60), 3)        # red = predicted
    return g, gt, pr, ov


# ---------------- FIG 1 : segmentation quality ----------------
q = d[d["oof_dice"].notna()].sort_values("oof_dice").reset_index(drop=True)
picks = [("WORST",  q.iloc[0]),          ("poor",   q.iloc[len(q) // 20]),
         ("median", q.iloc[len(q) // 2]), ("good",  q.iloc[-len(q) // 20]),
         ("BEST",   q.iloc[-1])]
fig, ax = plt.subplots(len(picks), 4, figsize=(15, 3.6 * len(picks)))
for i, (tag, r) in enumerate(picks):
    out = panel(r["img"], r["msk"], r["pred"])
    if out is None: continue
    g, gt, pr, ov = out
    for j, (im_, t) in enumerate([(g, "input (CLAHE)"), (gt, "ground truth"),
                                  (pr, "predicted"), (ov, "overlay")]):
        a = ax[i, j]; a.axis("off")
        a.imshow(im_, cmap="gray" if j < 3 else None)
        if i == 0: a.set_title(t, fontsize=11)
    ax[i, 0].set_ylabel(tag)
    ax[i, 3].set_title("%s  Dice %.3f  BI-RADS %s" %
                       (tag, r["oof_dice"],
                        int(r["assessment"]) if pd.notna(r["assessment"]) else "?"), fontsize=10)
plt.suptitle("MASS SEGMENTATION — green = radiologist, red = predicted (DS-Attn-UNet+ASPP)",
             fontsize=13, y=0.995)
plt.tight_layout()
plt.savefig(os.path.join(FIG, "1_segmentation_examples.png"), dpi=120, bbox_inches="tight")
plt.close()
print("saved 1_segmentation_examples.png")

# ---------------- FIG 2 : classification examples ----------------
if d["prob"].notna().any():
    L = (d.dropna(subset=["prob"])
           .groupby("lesion_key")
           .agg(y=("label", "max"), p=("prob", "mean")).reset_index())
    grid = np.linspace(0.05, 0.95, 181)
    accs = [( (L.p > t).astype(int) == L.y ).mean() for t in grid]
    THR = float(grid[int(np.argmax(accs))])
    print("threshold used for the picture: %.2f" % THR)

    dd = d.dropna(subset=["prob"]).copy()
    dd["case"] = np.where((dd.label == 1) & (dd.prob > THR), "TP",
                  np.where((dd.label == 0) & (dd.prob <= THR), "TN",
                  np.where((dd.label == 0) & (dd.prob > THR), "FP", "FN")))
    fig, ax = plt.subplots(4, 4, figsize=(15, 15))
    for i, cs in enumerate(["TP", "TN", "FP", "FN"]):
        sel = dd[dd.case == cs]
        sel = (sel.reindex(sel.prob.sub(THR).abs().sort_values(
                   ascending=(cs in ["FP", "FN"])).index).head(4))
        for j in range(4):
            a = ax[i, j]; a.axis("off")
            if j >= len(sel): continue
            r = sel.iloc[j]
            out = panel(r["img"], r["msk"], r["pred"])
            if out is None: continue
            a.imshow(out[3])
            a.set_title("%s   p(malig)=%.2f\ntruth=%s  BI-RADS %s"
                        % (cs, r["prob"], "MALIGNANT" if r["label"] == 1 else "benign",
                           int(r["assessment"]) if pd.notna(r["assessment"]) else "?"),
                        fontsize=9,
                        color={"TP": "green", "TN": "green", "FP": "red", "FN": "darkred"}[cs])
    plt.suptitle("MASS CLASSIFICATION — TP / TN / FP / FN  (green=truth outline, red=predicted)",
                 fontsize=13, y=0.995)
    plt.tight_layout()
    plt.savefig(os.path.join(FIG, "2_classification_examples.png"), dpi=110, bbox_inches="tight")
    plt.close()
    print("saved 2_classification_examples.png")

    # ---------------- FIG 3 : summary plots ----------------
    fig, ax = plt.subplots(1, 3, figsize=(17, 4.8))
    ax[0].hist(d["oof_dice"].dropna(), bins=40, color="#1f6fb4", edgecolor="white")
    ax[0].axvline(d["oof_dice"].mean(), color="red", ls="--",
                  label="mean %.3f" % d["oof_dice"].mean())
    ax[0].axvline(0.792, color="green", ls=":", lw=2,
                  label="annotation ceiling 0.792")
    ax[0].set_xlabel("Dice"); ax[0].set_ylabel("lesions")
    ax[0].set_title("Segmentation quality"); ax[0].legend(fontsize=9); ax[0].grid(alpha=.25)

    fpr, tpr, _ = roc_curve(L.y, L.p)
    ax[1].plot(fpr, tpr, lw=2, color="#1f6fb4",
               label="AUC = %.4f" % roc_auc_score(L.y, L.p))
    ax[1].plot([0, 1], [0, 1], "k--", lw=1)
    ax[1].set_xlabel("1 - specificity"); ax[1].set_ylabel("sensitivity")
    ax[1].set_title("Classification ROC (per lesion)"); ax[1].legend(); ax[1].grid(alpha=.25)

    cm = confusion_matrix(L.y, (L.p > THR).astype(int), labels=[0, 1])
    ax[2].imshow(cm, cmap="Blues")
    for (r_, c_), v in np.ndenumerate(cm):
        ax[2].text(c_, r_, str(v), ha="center", va="center", fontsize=16,
                   color="white" if v > cm.max() / 2 else "black")
    ax[2].set_xticks([0, 1]); ax[2].set_xticklabels(["pred benign", "pred malignant"])
    ax[2].set_yticks([0, 1]); ax[2].set_yticklabels(["true benign", "true malignant"])
    ax[2].set_title("Confusion matrix  (acc %.1f%%)"
                    % (100 * ((L.p > THR).astype(int) == L.y).mean()))
    plt.tight_layout()
    plt.savefig(os.path.join(FIG, "3_summary.png"), dpi=130, bbox_inches="tight")
    plt.close()
    print("saved 3_summary.png")

print("\nfigures in:", FIG)
```

---

## CELL 9 — Literature comparison

> I verified these numbers directly from the papers' abstracts.

```python
# ══════════════════════════════════════════════════════════════════════
# CELL 9 — MASS: comparison with published CBIS-DDSM studies
#   Numbers marked VERIFIED were read from the paper abstract.
#   Numbers marked CHECK must be confirmed against the original paper.
# ══════════════════════════════════════════════════════════════════════
import os, numpy as np, pandas as pd
import matplotlib; matplotlib.use("Agg")
import matplotlib.pyplot as plt
from matplotlib.patches import Patch

D   = "/root/autodl-tmp/CBIS"
FIG = os.path.join(D, "figures", "mass_final"); os.makedirs(FIG, exist_ok=True)

MY_DICE, MY_AUC, MY_ACC = 0.8998, 0.9022, 84.6
CEILING = 0.792          # CBIS-DDSM paper: masks vs radiologist (Lee et al. 2017)

# split: 2 = patient/case-level stated | 1 = unclear | 0 = this work
SEG = [
 ("THIS WORK (DS-Attn-UNet+ASPP)", MY_DICE, 0, "patient-grouped 5-fold CV",   "—"),
 ("Jalalian 2025 (UNet++)",        0.9292,  1, "not stated",                  "VERIFIED"),
 ("Ma 2023 (cross-view VAE)",      0.9246,  1, "not stated",                  "VERIFIED"),
 ("Ribeiro 2022 (MDA-Net)",        0.9025,  1, "benchmark, 1692 mammograms",  "VERIFIED"),
 ("Ribeiro 2022 (DynUNet)",        0.8967,  1, "benchmark, 1692 mammograms",  "VERIFIED"),
 ("Baccouche 2021 (Connected-UNets)", 0.8952, 1, "not stated",                "VERIFIED"),
 ("El-Banby 2024 (deep U-Net)",    0.8798,  1, "not stated",                  "VERIFIED"),
 ("Rajalakshmi 2020 (DS U-Net+CRF)", 0.8290, 1, "not stated",                 "VERIFIED"),
 ("Tsochatzidis 2021",             0.7220,  2, "official CBIS split",         "CHECK"),
]

CLS = [
 ("THIS WORK (DenseNet-121, predicted masks)", MY_AUC, MY_ACC, 0, "patient-grouped 5-fold CV", "—"),
 ("Ma 2023 (cross-view VAE)",      0.9320, None, 1, "not stated",             "VERIFIED"),
 ("Tsochatzidis 2021",             0.8620, 74.9, 2, "official CBIS split",    "CHECK"),
 ("Tiryaki 2023",                  0.8188, 76.2, 1, "not stated",             "CHECK"),
 ("Salama 2021",                   None,   82.5, 1, "CBIS subset; best result was on DDSM", "CHECK"),
]

TIER = {0: "THIS WORK", 1: "split not stated", 2: "patient/case-level stated"}
COL  = {0: "#1f6fb4", 1: "#b0aea6", 2: "#4a9d5f"}

W = 104
print("=" * W); print("TABLE 1 — MASS SEGMENTATION on CBIS-DDSM"); print("=" * W)
print("%-36s %-8s %-30s %s" % ("Study", "Dice", "Split protocol", "Source"))
print("-" * W)
for n, v, t, s, src in sorted(SEG, key=lambda r: -r[1]):
    print("%-36s %-8.4f %-30s %s%s" % (n, v, s, src, "   <<<" if t == 0 else ""))
print("-" * W)
print("CBIS-DDSM reference masks vs radiologist (Lee 2017): Dice %.3f" % CEILING)
print("  -> any Dice above ~0.79 is fitting the curators' automated algorithm,")
print("     not a radiologist's boundary. THIS WORK = %.4f." % MY_DICE)
print("=" * W)

print()
print("=" * W); print("TABLE 2 — MASS CLASSIFICATION on CBIS-DDSM"); print("=" * W)
print("%-44s %-8s %-8s %-28s %s" % ("Study", "AUC", "Acc", "Split protocol", "Source"))
print("-" * W)
for n, a, ac, t, s, src in sorted(CLS, key=lambda r: -(r[1] or 0)):
    print("%-44s %-8s %-8s %-28s %s%s"
          % (n, ("%.4f" % a) if a else "—", ("%.1f%%" % ac) if ac else "—",
             s, src, "   <<<" if t == 0 else ""))
print("=" * W)

# ---------------- figure ----------------
fig, (a1, a2) = plt.subplots(1, 2, figsize=(16, 6))

rows = sorted(SEG, key=lambda r: r[1])
nm = [r[0] for r in rows]; vl = [r[1] for r in rows]; tr = [r[2] for r in rows]
b = a1.barh(range(len(vl)), vl, height=.68, color=[COL[t] for t in tr],
            edgecolor=["black" if t == 0 else "none" for t in tr],
            linewidth=[1.8 if t == 0 else 0 for t in tr])
a1.axvline(CEILING, color="darkgreen", ls="--", lw=2)
a1.text(CEILING - 0.005, len(vl) - 0.4, "annotation ceiling 0.792", rotation=90,
        va="top", ha="right", fontsize=9, color="darkgreen")
a1.set_yticks(range(len(nm))); a1.set_yticklabels(nm, fontsize=9)
a1.set_xlim(0.65, 0.98); a1.set_xlabel("Dice coefficient")
a1.set_title("Mass segmentation on CBIS-DDSM", fontsize=12)
a1.grid(axis="x", alpha=.25); a1.set_axisbelow(True)
for bb, v in zip(b, vl):
    a1.text(v + .003, bb.get_y() + bb.get_height() / 2, "%.4f" % v, va="center", fontsize=8.5)

crow = [r for r in sorted(CLS, key=lambda z: (z[1] or 0)) if r[1] is not None]
nm2 = [r[0] for r in crow]; vl2 = [r[1] for r in crow]; tr2 = [r[3] for r in crow]
b2 = a2.barh(range(len(vl2)), vl2, height=.55, color=[COL[t] for t in tr2],
             edgecolor=["black" if t == 0 else "none" for t in tr2],
             linewidth=[1.8 if t == 0 else 0 for t in tr2])
a2.set_yticks(range(len(nm2))); a2.set_yticklabels(nm2, fontsize=9)
a2.set_xlim(0.75, 0.98); a2.set_xlabel("AUC")
a2.set_title("Mass classification on CBIS-DDSM", fontsize=12)
a2.grid(axis="x", alpha=.25); a2.set_axisbelow(True)
for bb, v in zip(b2, vl2):
    a2.text(v + .003, bb.get_y() + bb.get_height() / 2, "%.4f" % v, va="center", fontsize=8.5)

a1.legend(handles=[Patch(facecolor=COL[0], edgecolor="black", label="This work"),
                   Patch(facecolor=COL[2], label="patient/case-level split stated"),
                   Patch(facecolor=COL[1], label="split protocol not stated")],
          loc="lower right", fontsize=9)
plt.tight_layout()
plt.savefig(os.path.join(FIG, "4_literature_comparison.png"), dpi=140, bbox_inches="tight")
plt.close()
print("\nsaved 4_literature_comparison.png")
```

---

## The argument to make to the supervisor

Dice of 0.8998 sits mid-table numerically — but the framing matters more than the rank.

1. **The annotation ceiling.** The CBIS-DDSM descriptor paper measured its own
   masks against a radiologist's hand-drawn outlines at Dice 0.792 ± 0.108.
   Studies reporting 0.92–0.95 are matching the curators' level-set algorithm
   more closely than that algorithm matches a human. That is a defensible,
   citable critique.

2. **Split protocol.** Of the papers verified, none state a patient-level split.
   Baccouche 2021, Jalalian 2025, El-Banby 2024 and Ma 2023 all report on
   CBIS-DDSM without specifying that a patient's CC and MLO views stayed
   together. Since each lesion appears twice, a random split leaks almost every
   lesion across train and test.

3. **A useful anchor.** Ribeiro et al. 2022 benchmarked six architectures on
   1,692 CBIS-DDSM mammograms — almost exactly this work's 1,696 — and got Dice
   0.86–0.90, with the best (MDA-Net) at 0.9025. 0.8998 is level with the best
   of that benchmark, under a stricter protocol.

4. **Closest architectural relative.** Rajalakshmi et al. 2020 used a deeply
   supervised U-Net with CLAHE preprocessing — essentially Stage 1 without
   attention gates or ASPP — and reached 0.829 on CBIS-DDSM. This work is +0.07
   above the nearest published equivalent.

5. **Two-stage comparator.** Ma et al. 2023 reports both stages (Dice 0.9246,
   AUC 0.9320) but does not state its split — and does not run the
   reference-vs-predicted mask ablation done here. The controlled result
   (0.8441 → 0.8449 when swapping radiologist masks for predicted ones) is a
   contribution most of these papers do not offer.

### Paper links as given

- Baccouche 2021 — https://consensus.app/papers/details/3d01f245d68d5a3baf9084f00db4d7c8/
- Jalalian 2025 — https://consensus.app/papers/details/0e6988feb21450c18ff04952def1c1b3/
- El-Banby 2024 — https://consensus.app/papers/details/84ebd8fd85f25d2882428bdcf2cfa70d/
- Ma 2023 — https://consensus.app/papers/details/759072d7bdeb58e0a3042bdbdb024fad/
- Ribeiro 2022 — https://consensus.app/papers/details/6c418f97d0b85fbab10e796b8846416e/
- Rajalakshmi 2020 — https://consensus.app/papers/details/2b7dc7bb15cc5407bdf4f216ffc385d9/

### Open before submitting

The rows marked `CHECK` — **Tsochatzidis 2021, Tiryaki 2023, Salama 2021** —
came from an earlier literature review and could not be confirmed from
abstracts. Note that Salama's headline 98.87% is reported on **DDSM, not
CBIS-DDSM** — an easy row to get wrong.

---

## Decision recorded: stop on mass

> Yes. Nothing is broken. Remaining gains are +0.005–0.01 each from more seeds
> or a second backbone — hours of compute for a marginal change that won't alter
> your standing in that table. Your calcification stage, by contrast, has a
> genuine unsolved problem and a real novelty available. That's the better use
> of your remaining time.

---

## Infrastructure notes from the same exchange

- **The real work is not in this repository.** It lives on the AutoDL server at
  `/root/autodl-tmp/CBIS` — folds (`unified_folds_mass.csv`), predicted masks
  (`predmasks_mass/`), out-of-fold probabilities (`cv_mass_v2_oof.csv`) and
  figures (`figures/mass_final/`).
- **Why the branch was never pushed.** The commit `d9cf194` existed locally with
  the correct author identity, but every push returned **HTTP 403**: the Claude
  GitHub App lacks write access to `Masters-Research-`. Grantable at
  https://github.com/settings/installations
- That commit only reorganized the repo into `Images/`, `Notebooks/` and the CSV
  folder. No research results depended on it.
