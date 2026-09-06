# Transcript excerpt 3 — ensemble provenance, and how the session died

Recovered by paste from session `session_0166gYKz7AQzqvsQgMFn5pxe`.
Continues from excerpt 02. This is the final stretch of that conversation and
includes the error messages that ended it.

Verbatim as pasted. Nothing here has been re-run or re-verified.

---

## The push failure, diagnosed

> Diagnosed the push failure — it isn't a network problem and retrying won't fix it.

The clone's remote pointed at `github.com/Bushraali12/Masters-Research-`
(hyphens, trailing dash), which did not exist. The actual repositories:

- **`Bushraali12/Masters_Research`** — underscore, Jupyter Notebook, described
  "on CBIS-DDSM and INbreast dataset". The real one.
- **`Bushraali12/Masters_Research_`** — created a minute earlier the same day,
  a false start.

The 404 was GitHub correctly reporting a nonexistent repo; authentication was
fine (it resolved the user as `Bushraali12`). The session's GitHub scope was also
pinned to the nonexistent hyphenated name, so `Masters_Research` returned "not
configured for this session" as well.

**What the unpushed commit contained:** 30 files, pure renames, zero line
changes — moving image folders, CSVs and notebooks into `Notebooks/` and similar.
The reorganization commit, redundant with work the author had already done
manually. Recommendation given: drop it with `git reset --hard origin/main`
rather than repointing the remote and pushing into a repository the session was
not granted access to.

> None of this affects the paper. The manuscript files are delivered and live
> outside the repo.

---

## The blocking item: Cell A1

> Cell A1 is above, ready to run. It's the blocking item — Section 5.4 of the
> paper still carries pre-audit ablation numbers and can't be posted until you
> re-run it on the clean seven-model ensemble. Check that the FULL pooled AUC
> prints 0.9078; if it does, paste me the subgroup table and I'll rewrite that
> section around it.

Cell A1 itself was not captured in the paste — only its purpose and its
acceptance criterion (FULL pooled AUC = 0.9078).

---

## The canonical ensemble number, and the file-cluster problem

> Found it. Look at `mass_lesion_errors_clean.csv`: pooled **0.9078** — and 0.9078
> is exactly the figure in your earlier comparison-tables document. The 0.9090 in
> my handoff notes was transcription drift. Your real ensemble number was never
> lost.

But three sibling files sit in that cluster and disagree:

| File | Number |
|---|---|
| `mass_lesion_errors_clean.csv` | **0.9078** pooled |
| `mass_lesion_errors_clean.csv` (alt reading) | 0.9116 |
| `mass_lesion_errors_final.csv` | 0.9103 |
| `mass_lesion_errors_real.csv` | 0.9145 fold-averaged |

> I am not going to pick the highest one and call it your result.

The resolution is to rebuild the ensemble from the documented recipe so the
number comes from primary artifacts rather than a guessed-at file.

### The documented per-fold selection

```
SEL = {0: ["twostream", "endtoend_fused", "twostream_calcpre"],
       1: ["twostream", "endtoend_fused", "twostream_calcpre"],
       2: ["twostream", "endtoend_fused", "twostream_calcpre"],
       3: ["twostream", "endtoend_fused", "twostream_calcpre"],
       4: ["endtoend_fused", "twostream"]}
```

### Ensemble rebuild cell

```python
# ══════ rebuild the ensemble from the documented selection ══════
import pandas as pd, numpy as np, glob, os
from sklearn.metrics import roc_auc_score
from scipy.stats import rankdata

f = pd.read_csv("CBIS/unified_folds_mass.csv")
role = sorted([c for c in f.columns if c.lower().startswith("role_f")])
M = np.stack([f[c].astype(str).str.strip().str.lower().eq("test").values for c in role], 1)
f["_fold"] = M.argmax(1)
fmap = f.set_index("img")["_fold"].to_dict()
lmap = pd.read_csv("CBIS/mass_handcrafted_features.csv").set_index("img")["lesion_key"].to_dict()

SEL = {0: ["twostream", "endtoend_fused", "twostream_calcpre"],
       1: ["twostream", "endtoend_fused", "twostream_calcpre"],
       2: ["twostream", "endtoend_fused", "twostream_calcpre"],
       3: ["twostream", "endtoend_fused", "twostream_calcpre"],
       4: ["endtoend_fused", "twostream"]}

P = {}
for m in {x for v in SEL.values() for x in v}:
    d = pd.read_csv(f"CBIS/cv_mass_{m}_oof.csv")
    d["_f"] = d["img"].map(fmap); d["_lk"] = d["img"].map(lmap)
    d = d.dropna(subset=["_f", "_lk"])
    d["_r"] = rankdata(d["prob"]) / len(d)          # rank-normalise once across cohort
    P[m] = d.set_index("img")[["_f", "_lk", "true", "_r"]]
    print(f"  {m:<22} {len(d)} rows")

parts = []
for fold, models in SEL.items():
    idx = P[models[0]].index[P[models[0]]["_f"] == fold]
    r = np.mean([P[m].loc[idx, "_r"].values for m in models], axis=0)
    parts.append(pd.DataFrame(dict(img=idx, fold=fold, lk=P[models[0]].loc[idx, "_lk"].values,
                                   true=P[models[0]].loc[idx, "true"].values, p=r)))
E = pd.concat(parts)
L = E.groupby(["lk", "fold"], as_index=False).agg(true=("true", "max"), p=("p", "mean"))

reg = [roc_auc_score(g["true"], g["p"]) for _, g in E.groupby("fold")]
les = [roc_auc_score(g["true"], g["p"]) for _, g in L.groupby("fold")]
print(f"\nREBUILT ENSEMBLE  ({E.img.nunique()} regions -> {L.lk.nunique()} lesions)")
print(f"  region  pooled {roc_auc_score(E.true, E.p):.4f}   fold_mean {np.mean(reg):.4f} ± {np.std(reg, ddof=1):.4f}")
print(f"  lesion  pooled {roc_auc_score(L.true, L.p):.4f}   fold_mean {np.mean(les):.4f} ± {np.std(les, ddof=1):.4f}")
print("  per fold:", [round(x, 4) for x in les])

# how close are the three archive files to this rebuild?
print("\nAGREEMENT WITH ARCHIVE FILES")
for fp in ["CBIS/mass_lesion_errors_clean.csv", "CBIS/mass_lesion_errors_final.csv",
           "CBIS/mass_lesion_errors_real.csv", "CBIS/mass_ensemble_v3.csv"]:
    if not os.path.exists(fp): continue
    a = pd.read_csv(fp)
    kc = "lesion_key" if "lesion_key" in a.columns else None
    if kc is None or "p" not in a.columns: continue
    j = L.merge(a[[kc, "p"]].rename(columns={kc: "lk", "p": "p_arch"}), on="lk", how="inner")
    print(f"  {os.path.basename(fp):<32} n={len(j):<5} spearman={j['p'].corr(j['p_arch'], method='spearman'):.4f}")
```

> Whichever archive file correlates ~1.0 with the rebuild is the canonical one —
> and the rebuilt number itself is reproducible from primary OOF files, which is
> what you want in a thesis.

The last action before the session died was an edit to `ablation_AB.js`
(+59 −30), updating the document with the completed lesion-level table.

---

## How the session died

The failure sequence, verbatim:

```
An API error occurred
Failed to start Claude Code: chdir /home/user/Masters-Research-: no such file or directory

Try again
```

The author then wrote:

> The repo Masters-Research- was deleted and has now been restored. Re-clone it
> and continue on branch claude/organize-csv-notebooks-aoqtuc. From now on, push
> all work to Bushraali12/Masters_Research instead.

And it still failed:

```
An API error occurred
Try sending your message again.
An error occurred while executing Claude Code. You can try again by sending a new session.
An API error occurred
Try sending your message again.
Failed to start Claude Code: chdir /home/user/Masters-Research-: no such file or directory
```

### Cause

The session's environment was pinned to `Bushraali12/Masters-Research-` as its
source. That repository was **deleted**. On every container restart the harness
tries to clone it, the clone fails, the working directory
`/home/user/Masters-Research-` is never created, and `chdir` into it fails before
Claude Code can start. Every user-visible "An API error occurred" is that startup
failure, not a model or network error.

This supersedes two earlier guesses made during recovery: oversized conversation
state, and a usage limit. The "Session limit reached" seen in excerpt 02 was a
separate, transient event.

### Why restoring the repo did not fix it

Verified from this session on 2026-09-02: `Bushraali12/Masters-Research-` now
resolves and is readable anonymously (`main` at `7c9d46ad`). The restore worked.

Restoring a deleted GitHub repository does **not** restore the GitHub App's
repository-access selection. The session clones with app credentials, so an
authenticated clone still 404s even though anonymous read succeeds. Re-granting
access at https://github.com/settings/installations is the one fix worth trying.

Even if that works, the session remains pinned to the hyphenated repo as its
working directory, while the author's own instruction was to move all work to
`Bushraali12/Masters_Research`.

### Repository map, verified 2026-09-02

| Repo | State |
|---|---|
| `Bushraali12/Masters_Research` | **live and correct** — `main` at `7f6b35b`, recovery branch pushed |
| `Bushraali12/Masters-Research-` | restored, `main` at `7c9d46ad` — the dead session's pinned source |
| `Bushraali12/Masters_Research_` | resolves to nothing; the false start |
