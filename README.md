# H → Zγ* → μμγ (Dalitz) — Run-3 Analysis

Muon channel of the rare Higgs Dalitz decay. This guide runs the full chain from
NanoAOD to the combined limit and significance, and shows how to add more years.

The chain spans **three working areas**, each with its own environment:

1. **NanoBridge** — NanoAOD → mini-trees (Run-3 CMSSW area).
2. **skimTree + xAna** — skim + event selection (conda analysis env).
3. **HLLGFinalFits** — Tree2WS → Background → Signal → Datacard → Combine (SL7 container).

Chain order:
**NanoBridge → skimTree → xAna → Tree2WS → Background → Signal → Datacard → Combine.**

Years stay separate until the **Datacard** step, where each year enters as a separate
signal process; data is already combined at Tree2WS. Combine then produces the final
multi-year result.

---

## 1. NanoBridge — produce mini-trees

```bash
cd <path>/CMSSW_15_0_19/src/HiggsDalitz/NanoBridge/python
cmsenv
```

Choose the producer variant (copy it to `xAnaProducer_fixed.py`, which the runner uses):

```bash
cp xAnaProducer_fixed_v5.py xAnaProducer_fixed.py   # DATA  (applies HLT + object cuts)
cp xAnaProducer_fixed_v6.py xAnaProducer_fixed.py   # SIGNAL/MC (keeps all events for normalization)
```

Run locally (one file per process):

```bash
python3 run_xAnaProducerMultiFiles_fixed.py <input_list.txt> <output_dir> <data|mc> <year>
```

Or submit to HTCondor (one job per file):

```bash
# data (DAS datasets) — one .sh per year, or all years:
./submit_condor_2024.sh          # (each line: submit_condor.py <DAS> <ERA> <YEAR> data)
./submit_condor_all.sh           # all years

# Signal MC (private, local file list)
./submit_signal.sh

# Monitor / resubmit
python3 check_status.py <output_directory>
python3 resubmit_failed.py
```

Per-era corrections (photon scale&smearing, golden JSON) are dicts at the top of
`run_xAnaProducerMultiFiles_fixed.py`, keyed by `<year>`. Add a new era there.

**`<year>`:** `2022`, `2022EE`, `2023`, `2023BPix`, `2024`, `2025`, `2026`.
Sub-eras merge into the parent year downstream via `{year}*` wildcards.

 **`<era>`:** here contains "2022C" etc, just is used for the output sub-directory creation and nothing else.

---

## 2. skimTree — skim + MC-truth cut

```bash
cd <path>/HDalitzMu_Run3
source <path>/HDalitzEle/env_ana.sh
```

Compile and run per year (applies `diLepMCMass < 60 && HasIntPho == 0`):

```bash
g++ skimTree.cpp -o ./skimTree -I$CONDA_PREFIX/include -std=c++17 -Wall -O3 $(root-config --glibs --cflags --libs)
./skimTree 2024
./skimTree 2025
./skimTree 2026
```

Per-year input/output paths are vectors inside `skimTree.cpp` (`inpath_<year>`,
`outpath_<year>`, `ssCorrDir_<year>`). To add a year, copy a block and change the year.

---

## 3. xAna — event selection, weights, categories

Same environment as skimTree (`source .../env_ana.sh`). Run per year, data and MC
separately:

```bash
# Data
root -l -b runAna_Data_2024.C
root -l -b runAna_Data_2025.C
root -l -b runAna_Data_2026.C

# MC (signal)
root -l -b runAna_MC_2024.C
root -l -b runAna_MC_2025.C
root -l -b runAna_MC_2026.C
```

Each `runAna_MC_<year>.C` sets that year's **luminosity** and era string. To add a
year, copy one and change the era string, luminosity, and paths.

Produce the AMS / resolution / yield table (independent cross-check).
Argument `0` = MuPho categories, `1` = IsoMu categories:

```bash
root -l -b -q runSignificance_Run3.C\(0\)
```

---

## 4. Tree2WS — build workspaces

```bash
cmssw-el7 --bind /data3:/data3
cd <path>/CMSSW_11_3_4/src
cmsenv
cd <path>/HLLGFinalFits_Run3
source setup.sh
cd Tree2WS
```

Signal — run once per year:

```bash
python3 runTree2WS.py -s tree2ws -c config.py -y 2024
python3 runTree2WS.py -s tree2ws -c config.py -y 2025
python3 runTree2WS.py -s tree2ws -c config.py -y 2026
```

Data — run once (combined, no year):

```bash
python3 runTree2WS.py -s tree2ws_data -c config_data.py
```

`config.py`: set `inDir` to the xAna v6 signal output (picks up all years via glob).
`config_data.py`: add each year's data files to the flat `inputTreeFiles` list.

---

## 5. Background — data-driven fit

```bash
cd ../Background
make                       # only if bin/fTest_v2 is missing
python3 runBackground.py
```

Fits `WS/data_obs.root` (combined data) per category, writes the multipdf workspaces.

---

## 6. Signal — model + systematics

Run per year. `-ds` on `signalFit` is required (creates `NewSigPdf_*`):

```bash
cd ../Signal
for Y in 2024 2025 2026; do
  python3 runSignal.py -s calcShapeSyst -y $Y
  python3 runSignal.py -s calcYieldSyst -y $Y
  python3 runSignal.py -s signalFit     -y $Y -ds
done
```

Produces `WS/Interpolation/<year>/CMS_HLLG_Interp_*.root`. Keep all years present.

---

## 7. Datacard — combine years

```bash
cd ../Datacard
python3 makeYields.py
python3 makeDatacard.py
```

`makeYields.py` loops years and skips any without workspaces, so only existing years
enter — each as a separate signal process with its own luminosity.

---

## 8. Combine — limit & significance

```bash
cd ../Combine
cp ../Datacard/cards/*.txt cards/
cp ../Signal/WS/Interpolation/202*/CMS_HLLG_Interp_*.root cards/
cp ../Background/multipdf/CMS_HLLG_multipdf_13TeV_*.root cards/
python3 mergeCards.py
sh run_combine.sh
python3 makeLimitPlot.py
```

`mergeCards.py`: set `EXCLUDE_CATS = []` for the full category set.
`makeLimitPlot.py`: set the luminosity label to the combined total
(2024+2025+2026 ≈ 249 fb⁻¹).

---

## 9. Adding more years (e.g. 2022, 2023)

Repeat each stage for the new year; the combination is automatic once the workspaces
exist.

1. NanoBridge: add the era's correction entries; run signal (v6) and data (v5).
2. skimTree: add the year's vectors; recompile; `./skimTree <year>`.
3. xAna: make `runAna_MC_<year>.C` / `runAna_Data_<year>.C` (era string + luminosity); run.
4. `commonObjects.py`: ensure the year is in `years`/`yearsStr` and `lumiMap`; update `lumiMap["combined"]`.
5. Tree2WS: signal picks it up automatically; add its data files to `config_data.py` and re-run the data step.
6. Background: re-run `runBackground.py`.
7. Signal: run the 3-step sequence for the new year.
8. Datacard: re-run — the new year is picked up automatically.
9. Combine: re-copy the Datacard cards + all years' interpolation WS + bkg multipdf into `cards/`, re-run; update the lumi label.

---

## Notes

- v5 = data, v6 = signal/MC. Wrong variant on signal breaks the MC normalization.
- `-ds` is mandatory on `signalFit`; `calcShapeSyst`/`calcYieldSyst` must run first
  (and be re-run if the signal is reprocessed).
- Data Tree2WS is run once (combined); signal is per-year.
- The Combine copy step is manual — all years' interpolation WS + bkg multipdf must be
  in `cards/` before `mergeCards.py`.
- `makeModelPlot` (optional signal overlay plot) has no missing-year skip-guard yet;
  skip it or add the guard.
- The only photon-ID cut in the whole chain is the MVA cut in xAna: photons are kept
  if `MVA > -0.9` (i.e. `if (MVA < -0.9) continue;`), on the stored NanoAOD Winter22V1
  mvaID. WP90/WP80 flags are stored for future tightening.
