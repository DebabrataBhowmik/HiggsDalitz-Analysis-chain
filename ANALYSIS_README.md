# H → Zγ* → μμγ (Dalitz) — Run-3 Analysis: Full Processing Guide

This document describes the **complete analysis chain** for the Run-3 H → μμγ Dalitz
search, from NanoAOD to the final combined limit and significance. It is written so
that someone with **no prior knowledge** of this analysis (including a future you who
has forgotten the details) can reproduce it, extend it to new years, or combine
additional years (e.g. add 2022 and 2023 to the existing 2024+2025+2026).

It assumes you already have the code checked out (two repositories, see below) and
the software environments set up. It does **not** re-explain the physics; it is an
operational runbook.

---

## 0. Overview of the chain

```
NanoAOD  ──►  NanoBridge  ──►  skimTree  ──►  xAna  ──►  Tree2WS  ──►  Background
(central/     (produce         (skim +        (event       (build       (data-driven
 private)     nanoBridge)     MC-truth cut)   selection,   RooFit        bkg fit)
              output trees                    weights,     workspaces)
                                              categories)
                                                                          │
                                          ┌───────────────────────────────┘
                                          ▼
                                       Signal  ──►  Datacard  ──►  Combine  ──►  limit
                                       (signal      (per-year     (merge cats,   & p-value
                                        model +      cards,        run combine)   plots
                                        systs)       COMBINE       ← years merge here
                                                     years here)
```

**Where years stay separate vs. combine:**
* Separate per year: NanoBridge, skimTree, xAna, Tree2WS (signal), Signal model.
* **Data is combined across years already at Tree2WS** (single `data_obs`).
* **Signal years merge at the Datacard step** (each year enters as a separate signal
  process with its own luminosity). Combine then merges categories into the final
  multi-year likelihood.

---

## 1. Repositories and environments

**Two repositories:**

1. **NanoBridge** — NanoAOD → mini-trees.
   `https://github.com/DebabrataBhowmik/NanoBridge`
   Runs in a Run-3 CMSSW area (e.g. `CMSSW_15_0_19`), `cmsenv`.

2. **HLLGFinalFits** — skim → xAna → Tree2WS → Background → Signal → Datacard → Combine.
   Runs in an SL7 singularity container with an older CMSSW for combine:
   ```bash
   cmssw-el7 --bind /data3:/data3
   cd CMSSW_11_3_4/src
   cmsenv
   cd HLLGFinalFits_Run3
   source setup.sh
   ```
   (xAna itself runs in a conda env `hdalitzAna` — `source env_ana.sh` — outside the
   container.)

The two repos are deliberately separate: NanoBridge is a **generic** NanoAOD tool;
HLLGFinalFits is the analysis-specific fitting chain.

---

## 2. NanoBridge — produce mini-trees

Runs per (year, data/mc). **Two producer variants exist**; copy the one you want to
`xAnaProducer_fixed.py` before running (the runner uses that filename):

```bash
cp xAnaProducer_fixed_v5.py xAnaProducer_fixed.py   # DATA
cp xAnaProducer_fixed_v6.py xAnaProducer_fixed.py   # SIGNAL / MC
```

* **v5** applies HLT + object-count pre-selection → fewer events (fine for data,
  which is not normalized).
* **v6** keeps every event (no trigger/object gate). The total processed-event count
  is needed to normalize MC weights, so nothing may be dropped. Larger and slower,
  but manageable for MC. **Always use v6 for signal/MC**, or the normalization is wrong.

**Run locally, one file per process (avoids memory build-up):**
```bash
cd HiggsDalitz/NanoBridge/python
python3 run_xAnaProducerMultiFiles_fixed.py <input_list.txt> <output_dir> <data|mc> <year>
```

**Or submit to HTCondor (one job per file — preferred for large sets):**
```bash
# data (DAS datasets) — one .sh per year, or all years:
./submit_condor_2024.sh          # (each line: submit_condor.py <DAS> <ERA> <YEAR> data)
./submit_condor_all.sh

# signal MC (private, not on DAS — pass a local file list):
./submit_signal.sh               # (submit_condor.py <list.txt> <ERA> <YEAR> mc [OUTPUT_BASE])

# monitor / resubmit:
python3 check_status.py <output_directory>
python3 resubmit_failed.py       # after editing FAILED_LIST/MODE/YEAR inside it
```

**Per-era corrections** live in dicts at the top of `run_xAnaProducerMultiFiles_fixed.py`
(`GOLDEN_JSON` for data; `EGAMMA_SS_JSON` photon scale&smearing per era). To add a new
era, add its entries there — no producer code change. (2025/2026 reuse Summer-24-based
S&S files, matching the CMS plan.)

**`<year>` / campaign strings:** `2022`, `2022EE`, `2023`, `2023BPix`, `2024`, `2025`, `2026`.
Sub-eras merge into the parent year downstream via `{year}*` wildcards.

---

## 3. skimTree — skim + MC-truth selection

`skimTree.cpp` reads NanoBridge output and applies the Dalitz MC-truth cut
(`diLepMCMass < 60 && HasIntPho == 0`), writing skimmed trees.

Per-year input/output paths are hardcoded as vectors inside `skimTree.cpp`
(`inpath_<year>`, `outpath_<year>`, `ssCorrDir_<year>`), 18 entries each
(6 production modes × 3 masses: 120/125/130). To add a year, copy an existing block
and change the year in the paths.

```bash
g++ skimTree.cpp -o ./skimTree -I$CONDA_PREFIX/include -std=c++17 -Wall -O3 $(root-config --glibs --cflags --libs)
./skimTree 2024
./skimTree 2025
./skimTree 2026
```

`ssCorrDir_<year> = ""` → prints a "no SS correction" warning (expected for Run-3;
shower-shape corrections not yet available). The `phoCorrESEnToRawE doesn't exist`
message is harmless (xAna never reads that branch).

---

## 4. xAna — event selection, weights, categories

xAna applies the full offline selection (good vertex, HLT, muons, muon pair, photon,
kinematics), computes the MC weight `mcwei = xs * lumi / totalEvents`, assigns the
analysis category (R9/eta based), and writes per-year "mini-tree" files with
`CMS_higgs_mass`, `weight`, `category` (plus stored photon MVA / WP flags for study).

Run via the per-year runner macro `runAna_MC_<year>.C` (sets the year's **luminosity**
and era string). To add a new year, copy `runAna_MC_2024.C` and change:
* the era string (`"Run3_<year>"`),
* the integrated **luminosity** (must be the year's real lumi),
* the input/output paths.

Data and signal are run separately (data un-normalized, signal normalized by `mcwei`).

**Photon ID:** the only photon-ID cut in the entire chain is `MVA < -0.9` in xAna,
applied to the stored Run-3 NanoAOD Winter22V1 `Photon_mvaID` for both data and MC.
The `[WARN] No HggPhoID TMVA weights for era Run3_<year>` message is harmless — the
UL2018 TMVA reader is loaded but its output is bypassed. (Future work: move to an EGM
working point WP90/WP80, already stored, and enable EGM photon-ID scale factors.)

**Significances (optional cross-check):** `runSignificance_Run3.C` computes per-category
AMS from the mini-trees (independent of the combine chain).

---

## 5. HLLGFinalFits — enter the container

Everything below runs inside the SL7 container with combine:
```bash
cmssw-el7 --bind /data3:/data3
cd CMSSW_11_3_4/src && cmsenv
cd HLLGFinalFits_Run3 && source setup.sh
```

Per-year settings (luminosities, the active `years`/`yearsStr` list, categories) live
in **`commonObjects.py`**. `lumiMap` holds each year's lumi in fb⁻¹.

---

## 6. Tree2WS — build RooFit workspaces

```bash
cd Tree2WS

# SIGNAL — run once per year (writes WS/<year>/signal_*.root):
python3 runTree2WS.py -s tree2ws -c config.py -y 2024
python3 runTree2WS.py -s tree2ws -c config.py -y 2025
python3 runTree2WS.py -s tree2ws -c config.py -y 2026

# DATA — run ONCE, no year (writes a single combined WS/data_obs.root):
python3 runTree2WS.py -s tree2ws_data -c config_data.py
```

* `config.py` (signal): set `inDir` to the xAna **v6** signal output. It loops all
  years in `commonObjects` and globs `{inDir}/{year}*/Signal/...` — so new years are
  picked up automatically (no per-year edit).
* `config_data.py` (data): `inputTreeFiles` is a **flat list** — add each year's data
  mini-trees to it. Output is a single combined `data_obs.root` (no `{year}` in path),
  so the data step is run once and covers all listed years.

---

## 7. Background — data-driven fit

```bash
cd Background
make                       # only if bin/fTest_v2 doesn't exist yet
python3 runBackground.py   # no arguments
```

Reads `WS/data_obs.root` (the combined multi-year data), fits sidebands per category,
writes the multipdf workspaces `multipdf/CMS_HLLG_multipdf_13TeV_<cat>.root`.

---

## 8. Signal — signal model + systematics

Run **per year**. The three steps must run in order; `-ds` on `signalFit` is
**mandatory** (it creates `NewSigPdf_*`, which the datacard references — without it,
combine fails with "Object NewSigPdf does not exist").

```bash
cd Signal
for Y in 2024 2025 2026; do
  python3 runSignal.py -s calcShapeSyst -y $Y
  python3 runSignal.py -s calcYieldSyst -y $Y
  python3 runSignal.py -s signalFit     -y $Y -ds
done
```

Produces per-year interpolation workspaces `WS/Interpolation/<year>/CMS_HLLG_Interp_*.root`.
Keep all years' `WS/Interpolation/<year>/` present — the datacard reads them all.

`makeModelPlot` (the multi-year overlay plot) currently loops **all** years in
`commonObjects` with no skip-guard, so it errors on years without workspaces. Either
add a skip-guard (recommended) or skip this plot — it does not feed the datacard.

---

## 9. Datacard — where the years combine

```bash
cd Datacard
python3 makeYields.py      # no arguments
python3 makeDatacard.py    # no arguments
```

`makeYields.py` loops `yearsStr`, reading each year's interpolation WS. It has a
**skip-guard**: years whose `WS/Interpolation/<year>/` directory is absent are skipped
with no error. So with `years = [2022,2023,2024,2025,2026]` it automatically uses only
the years that exist (currently 2024/2025/2026). Each year enters as a separate signal
process with rate `lumiMap[year] * 1000`.

`bbH` in VBF categories may print `yields < 0, discarded` — harmless.

---

## 10. Combine — final limit & significance

```bash
cd Combine

# copy ALL available years' interpolation WS + the bkg multipdf WS into cards/
cp ../Signal/WS/Interpolation/202*/CMS_HLLG_Interp_*.root cards/
cp ../Background/multipdf/CMS_HLLG_multipdf_13TeV_*.root cards/

python3 mergeCards.py      # merges per-category cards (respects EXCLUDE_CATS)
sh run_combine.sh
python3 makeLimitPlot.py
```

* `mergeCards.py` has `EXCLUDE_CATS` — set to `[]` for the full category set, or list
  categories to drop (e.g. one with a pathological background fit).
* `makeLimitPlot.py` — update the luminosity label to the **sum of the combined years**
  (e.g. 2024+2025+2026 = 109.95+110.84+28.06 ≈ 249 fb⁻¹). The limit math itself uses
  the per-year `lumiMap` rates automatically; only the plot label is manual.

The copy step is manual because the datacard references the WS by bare filename, so
combine needs them present in `cards/`.

---

## 11. Adding new years (e.g. combine 2022 + 2023 with 2024/25/26)

To bring 2022 and 2023 into the combination, repeat the per-year work for each new year;
the merge is automatic once their workspaces exist.

1. **NanoBridge:** ensure `EGAMMA_SS_JSON` / `GOLDEN_JSON` have 2022/2023 (and 2022EE/
   2023BPix sub-eras) entries. Run signal (v6) and data (v5) for each.
2. **skimTree:** add `inpath_2022/2023`, `outpath_*`, `ssCorrDir_*` vectors and dispatch;
   recompile; `./skimTree 2022` etc.
3. **xAna:** create `runAna_MC_2022.C` / `2023.C` with the correct era string and lumi;
   run data + signal.
4. **commonObjects.py:** confirm 2022/2023 are in `years`/`yearsStr` and `lumiMap` has
   their luminosities; update `lumiMap["combined"]` to the new total.
5. **Tree2WS:** signal `config.py` picks new years up automatically (year loop + glob).
   Add the new years' data files to `config_data.py`'s `inputTreeFiles`, then re-run the
   single data step so `data_obs` includes them.
6. **Background:** re-run `runBackground.py` (now over the enlarged combined data).
7. **Signal:** run the three-step sequence for 2022 and 2023 (`-ds` on signalFit).
8. **Datacard:** re-run `makeYields.py` + `makeDatacard.py` — the skip-guard now finds
   2022/2023 too, so they enter as new signal processes automatically.
9. **Combine:** the `202*` copy glob picks up the new interpolation WS; re-run
   `mergeCards.py` / `run_combine.sh` / `makeLimitPlot.py`; update the lumi label.

That is the whole point of the per-year structure: the combination is just "produce
each year's workspaces, then let the datacard loop pick up whatever exists."

---

## 12. Things to keep in mind (gotchas we hit)

* **v5 = data, v6 = signal/MC.** Copy the right producer to `xAnaProducer_fixed.py`
  before each NanoBridge run. Using v5 for signal breaks the MC normalization.
* **`-ds` is mandatory on `signalFit`** or combine can't find `NewSigPdf`.
* **`calcShapeSyst`/`calcYieldSyst` must run before `signalFit`** and must be re-run
  if the signal is reprocessed (stale `.pkl` files give wrong systematics).
* **Data Tree2WS is combined and run once** (no `-y`); signal is per-year.
* **Datacard skip-guard** lets `years` contain years you don't have — they're skipped.
  `makeModelPlot` lacks this guard (fix or skip it).
* **Combine copy step is manual** — all years' interpolation WS + bkg multipdf must be
  copied into `cards/` before `mergeCards.py`.
* **Update the lumi label** in `makeLimitPlot.py` to the combined total.
* **NanoBridge memory:** running many files in one process accumulates memory (PyROOT).
  Use one file per process (condor, or a shell loop) for large sets.
* **Normalization subtlety (why v6 exists):** the MC weight divides by the total
  processed events. If the producer pre-filters events (trigger/objects) before that
  count, the denominator is too small and the signal is inflated. v6 keeps all events
  so the denominator matches how acceptance × efficiency is defined — the same way the
  Run-2 ggNtuplizer stored all events and applied selection later.

---

## 13. Key paths (fill in / adjust for your setup)

* NanoBridge working dir: `.../CMSSW_15_0_19/src/HiggsDalitz/NanoBridge/python/`
* NanoBridge signal output (v6): `.../nanoBridgeOutput_v6/Signals/<year>/`
* skimTree output: `.../SkimTree_v6/<year>/`
* xAna signal mini-trees (v6): `.../xAnaOutput/v6/<year>/Signal/`
* xAna data mini-trees (v5): `.../xAnaOutput/<v5 path>/<year>/`
* HLLGFinalFits: `.../CMSSW_11_3_4/src/HLLGFinalFits_Run3/`
* Private signal NanoAOD: `/eos/cms/store/user/sqian/dalitz_nanoaod/merged/*.root`

---

*Chain order, one line:*
**NanoBridge → skimTree → xAna → Tree2WS (signal per-year, data once) → Background →
Signal (per-year, `-ds`) → Datacard (combines years) → Combine (limit + p-value).**
