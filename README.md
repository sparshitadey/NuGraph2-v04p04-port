# NuGraph2 Port: v10\_06\_00\_01p01 → v10\_06\_00\_04p04

> Port of the NuGraph2 GNN extensions from icaruscode **v10\_06\_00\_01p01** (1D deconvolution) to **v10\_06\_00\_04p04** (2D deconvolution / WireCell) for the ICARUS experiment.

S. Dey | May 2026

---

## Table of Contents

- [🗂️ Dev Area Structure](#-dev-area-structure)
- [🔬 Overview](#-overview)
- [⚡ What Changed](#-what-changed)
- [🌿 Repository Branches](#-repository-branches)
- [🏗️ Build Order](#-build-order)
- [🚀 Environment Setup](#-environment-setup)
- [🔧 Key Fixes and Changes](#-key-fixes-and-changes)
- [🔀 Pipeline: Before and After](#-pipeline-before-and-after)
- [📊 NuGraph2 Hit Filter Statistics](#-nugraph2-hit-filter-statistics)
- [🧠 NuGraph2 Model](#-nugraph2-model)
- [📝 Notes on lardataobj and larrecodnn](#-notes-on-lardataobj-and-larrecodnn)
- [📋 Pending PRs](#-pending-prs)
- [🛠️ For Developers](#-for-developers)
- [📬 Contact](#-contact)

---

## 🗂️ Dev Area Structure

The mrb dev area lives at `/exp/icarus/app/users/sdey2/dev/dev_v10_06_00_04p04_nugraph/`. The `srcs/` directory contains all seven checked-out packages. Key modified files are marked with `<--`.

```
dev_v10_06_00_04p04_nugraph/
|
+-- srcs/
|   |
|   +-- icaruscode/                           (branch: feature/sdey_NuGraph2_Filter_v04p04)
|   |   +-- icaruscode/TPC/NuGraph/
|   |   |   +-- nugraph_icarus.fcl            <-- NGMultiSliceCryoW hitInput fix
|   |   |   +-- ICARUSNuGraphInference_module.cc
|   |   |   +-- ICARUSNuGraphMultiLoader_tool.cc
|   |   |   +-- ICARUSFilteredNuSliceHitsProducer_module.cc
|   |   |   +-- ICARUSNuSliceHitsProducer_module.cc
|   |   +-- icaruscode/TPC/SignalProcessing/HitFinder/
|   |   |   +-- HitMerger_module.cc           <-- reserve() fix (pending PR)
|   |   +-- fcl/reco/Definitions/
|   |       +-- stage1_icarus_defs.fcl        <-- Pandora -> cluster3DCryoE switch
|   |
|   +-- larpandoracontent/                    (branch: feature/sdey_NuGraphInterface_v04p04)
|   |   +-- larpandoracontent/LArObjects/
|   |       +-- LArCaloHit.h                  <-- m_hitScores, m_hitScoreLabels added
|   |
|   +-- larpandora/                           (branch: feature/sdey_NuGraphInterface_v04p04)
|   |   +-- LArPandoraInterface/
|   |       +-- LArPandoraOutput.cxx          <-- NuGraph hit label collection
|   |
|   +-- sbnanaobj/                            (branch: feature/sdey_NuGraph2_MoveVars_v04p04)
|   |   +-- sbnanaobj/StandardRecord/
|   |       +-- SRNuGraphSliceInfo.h          <-- new class
|   |       +-- SRSlice.h                     <-- ng_plane[3] added
|   |       +-- classes_def.xml               <-- v10_00_05 checksums
|   |
|   +-- sbncode/                              (branch: feature/sdey_CAFMaker_PandoraAfterNuGraph_v04p04)
|   |   +-- sbncode/CAFMaker/
|   |       +-- CAFMakerParams.h              <-- NuGraphSlicesLabel (renamed)
|   |       +-- CAFMaker_module.cc            <-- art::FindOneP lookup
|   |       +-- FillReco.cxx                  <-- plane-by-plane semantic vars
|   |       +-- FillReco.h                    <-- updated signatures
|   |
|   +-- lardataobj/                           (branch: feature-nugraph-multislice, llena)
|   |   -- class definitions for NuGraph multi-slice data products
|   |
|   +-- larrecodnn/                           (branch: feature-nugraph-multislice, llena)
|   |   +-- larrecodnn/NuGraph/Tools/
|   |       +-- FilterDecoder_tool.cc         <-- art::Ptr key -> positional index fix
|   |
|   +-- CMakeLists.txt                        <-- correct build order (see Build Order)
|
+-- build_slf7.x86_64/                        (compiled binaries -- not in git)
+-- localProducts_larsoft_v10_06_00_02_e26_prof/  (generated -- not in git)
+-- portDebugLogs&FilesArchive/               (debug logs and FCL dumps)
```

---

## 🔬 Overview

NuGraph2 is a Graph Neural Network (GNN) that runs inside the ICARUS Stage 1 reconstruction pipeline. It assigns semantic labels (MIP, HIP, Shower, Michel, Diffuse) and a filter score to individual TPC hits, then feeds a second-pass Pandora reconstruction using only neutrino-tagged hits.

The NuGraph2 extensions were developed against icaruscode **v10\_06\_00\_01p01** (1D deconvolution / standard Gaussian hit finding). This port makes the same NuGraph2 functionality available in **v10\_06\_00\_04p04** (2D deconvolution / WireCell), which introduces per-TPC hit collections and a new `HitMerger` module.

The full port guide (detailed changes per file, common errors, checksums) is available in `for_developers/docs/` in this repository.

---

## ⚡ What Changed

The single most important change between the two base releases is the signal processing pipeline:

| | v10\_06\_00\_01p01 (1D) | v10\_06\_00\_04p04 (2D) |
|---|---|---|
| Hit finding | Gaussian 1D deconvolution | WireCell 2D deconvolution |
| Per-TPC hits | Single collection per cryostat | Separate collections per TPC half |
| Hit merger | Not present | `HitMerger` merges per-TPC hits |
| Pandora hit input | `gaushit` (single collection) | `cluster3DCryoE/W` (via this port) |
| NuGraph hit source | `combineHitsCryoE/W` (attempted) | `cluster3DCryoE/W` (correct) |
| NuGraph model | 1D-trained `model_mpvmpr_bnb_numu_cos.pt` | 2D-trained `model_mpvmpr_1201.pt` |

---

## 🌿 Repository Branches

Seven repositories carry the NuGraph2 changes. Use these exact branches:

| Repository | Fork | Branch |
|---|---|---|
| icaruscode | [sparshitadey/icaruscode](https://github.com/sparshitadey/icaruscode) | `feature/sdey_NuGraph2_Filter_v04p04` |
| larpandoracontent | [sparshitadey/larpandoracontent](https://github.com/sparshitadey/larpandoracontent) | `feature/sdey_NuGraphInterface_v04p04` |
| larpandora | [sparshitadey/larpandora](https://github.com/sparshitadey/larpandora) | `feature/sdey_NuGraphInterface_v04p04` |
| sbnanaobj | [sparshitadey/sbnanaobj](https://github.com/sparshitadey/sbnanaobj) | `feature/sdey_NuGraph2_MoveVars_v04p04` |
| sbncode | [sparshitadey/sbncode](https://github.com/sparshitadey/sbncode) | `feature/sdey_CAFMaker_PandoraAfterNuGraph_v04p04` |
| lardataobj | [sparshitadey/lardataobj](https://github.com/sparshitadey/lardataobj) | `feature-nugraph-multislice` |
| larrecodnn | [sparshitadey/larrecodnn](https://github.com/sparshitadey/larrecodnn) | `feature-nugraph-multislice` |

> `lardataobj` and `larrecodnn` track [leonardo-lena/lardataobj](https://github.com/leonardo-lena/lardataobj) and [leonardo-lena/larrecodnn](https://github.com/leonardo-lena/larrecodnn) respectively (branch `feature/nugraph-multislice`). No changes were made to these packages --- they are forked here for completeness and reproducibility.

---

## 🏗️ Build Order

CMake requires packages to be built in dependency order. Use this sequence:

```
1. larpandoracontent   (most upstream -- pure Pandora C++ additions)
2. lardataobj          (data object definitions used by larrecodnn)
3. larrecodnn          (base NuGraph2 inference framework)
4. larpandora          (LArSoft <-> Pandora interface)
5. sbnanaobj           (CAF data object definitions -- must precede sbncode)
6. icaruscode          (ICARUS-specific modules and FCL sequences)
7. sbncode             (CAF filling logic)
```

The `CMakeLists.txt` in this repository reflects the correct ordering. Copy it into `$MRB_SOURCE/` before building.

> ⚠️ `mrb uc` does not reliably produce the correct order. Edit `$MRB_SOURCE/CMakeLists.txt` manually to match the sequence above.

---

## 🚀 Environment Setup

### 1. Start the SL7 container and set up ICARUS

```bash
sh /exp/$(id -ng)/data/users/vito/podman/start_SL7dev_jsl.sh
source /cvmfs/icarus.opensciencegrid.org/products/icarus/setup_icarus.sh
htgettoken -a htvaultprod.fnal.gov -i icarus
```

### 2. Set up the dev area

```bash
setup icaruscode v10_06_00_04p04 -q e26:prof
cd /path/to/your/dev_area
source localProducts*/setup && mrbslp
```

### 3. Check out all branches

```bash
cd $MRB_SOURCE

# icaruscode
mrb g -t v10_06_00_04p04 icaruscode
cd icaruscode
git remote add sdey https://github.com/sparshitadey/icaruscode.git
git fetch sdey
git checkout -b feature/sdey_NuGraph2_Filter_v04p04 sdey/feature/sdey_NuGraph2_Filter_v04p04
cd ..

# larpandoracontent
mrb g -t v04_15_01 larpandoracontent
cd larpandoracontent
git remote add sdey https://github.com/sparshitadey/larpandoracontent.git
git fetch sdey
git checkout -b feature/sdey_NuGraphInterface_v04p04 sdey/feature/sdey_NuGraphInterface_v04p04
cd ..

# larpandora
mrb g -t v10_00_19 larpandora
cd larpandora
git remote add sdey https://github.com/sparshitadey/larpandora.git
git fetch sdey
git checkout -b feature/sdey_NuGraphInterface_v04p04 sdey/feature/sdey_NuGraphInterface_v04p04
cd ..

# sbnanaobj
mrb g -t v10_00_05 sbnanaobj
cd sbnanaobj
git remote add sdey https://github.com/sparshitadey/sbnanaobj.git
git fetch sdey
git checkout -b feature/sdey_NuGraph2_MoveVars_v04p04 sdey/feature/sdey_NuGraph2_MoveVars_v04p04
cd ..

# sbncode
mrb g -t v10_06_00_04 sbncode
cd sbncode
git remote add sdey https://github.com/sparshitadey/sbncode.git
git fetch sdey
git checkout -b feature/sdey_CAFMaker_PandoraAfterNuGraph_v04p04 sdey/feature/sdey_CAFMaker_PandoraAfterNuGraph_v04p04
cd ..

# lardataobj (Leonardo's branch -- no local changes)
mrb g -t v10_01_00 lardataobj
cd lardataobj
git remote add llena https://github.com/leonardo-lena/lardataobj.git
git fetch llena
git checkout -B feature-nugraph-multislice llena/feature/nugraph-multislice
cd ..

# larrecodnn (Leonardo's branch -- no local changes)
mrb g -t v10_01_10 larrecodnn
cd larrecodnn
git remote add llena https://github.com/leonardo-lena/larrecodnn.git
git fetch llena
git checkout -B feature-nugraph-multislice llena/feature/nugraph-multislice
cd ..
```

### 4. Set the 2D model path and build

```bash
export FW_SEARCH_PATH=/path/to/your/model/directory:$FW_SEARCH_PATH
cd $MRB_BUILDDIR
mrbsetenv
mrb i -j8 2>&1 | tee build.log
```

### 5. Run the NuGraph Stage 1

```bash
lar -c stage1_run2_icarus_overlay_NuGraphReco.fcl -n 5 \
  -s /path/to/stage0/input.root \
  --process-name redo \
  -o stage1_nugraph_output.root \
  > stage1_nugraph.log 2>&1
```

---

## 🔧 Key Fixes and Changes

### Fix 1: HitMerger vector reallocation bug ⚠️ pre-existing in base v04p04

**File:** `icaruscode/TPC/SignalProcessing/HitFinder/HitMerger_module.cc`

`HitMerger::produce()` stores raw `recob::Hit*` pointers as keys in `recobHitToPtrMap`. With two input labels (`gaushit2dTPCEW` and `gaushit2dTPCEE`), `emplace_back` on the second pass triggers reallocation of `outputHitPtrVec`, invalidating all pointers stored from the first pass. The `makeWireAssns`, `makeRawDigitAssns`, and `makeChanROIAssns` functions then dereference dangling pointers.

**Fix:** Pre-count all hits and call `reserve()` before the loop.

```cpp
// Pre-count total hits to reserve capacity (Sparshita Dey 2026)
size_t totalHits = 0;
for (const auto& inputTag : HitMergerfHitProducerLabelVec) {
    art::Handle<std::vector<recob::Hit>> hitHandle;
    evt.getByLabel(inputTag, hitHandle);
    if (hitHandle.isValid()) totalHits += hitHandle->size();
}
outputHitPtrVec->reserve(totalHits);
```

> This bug exists in vanilla v04p04 and affects all workflows using `HitMerger`. It should be submitted as a PR to base icaruscode.

---

### Fix 2: NGMultiSliceCryoW wrong hitInput (FCL bug)

**File:** `icaruscode/TPC/NuGraph/nugraph_icarus.fcl`

`NGMultiSliceCryoW` was defined as `@local::NGMultiSliceCryoE` and inherited `hitInput: "nuslhitsCryoE"` for its loader and decoder tools --- feeding the west cryostat NuGraph with east cryostat hits.

**Fix:** Add explicit overrides in all three FCL copies.

```
NGMultiSliceCryoW.LoaderTool.hitInput: "nuslhitsCryoW"
NGMultiSliceCryoW.LoaderTool.spsInput: "nuslhitsCryoW"
```

---

### Fix 3: Switch Pandora to cluster3DCryoE (architecture fix)

**File:** `icaruscode/fcl/reco/Definitions/stage1_icarus_defs.fcl`

In the initial port, Pandora read hits from `combineHitsCryoE/W` (produced by HitMerger from `gaushit2dTPCEW/EE`) while NuGraph's decoder tools read SpacePoints from `cluster3DCryoE/W` (produced from `gaushitPT2dTPCEW/EE`). These are different hit collections with different `art::Ptr` keys, causing a key mismatch crash in `FilterDecoder::writeToEvent`.

**Fix:** Point Pandora at `cluster3DCryoE/W` as its hit source, making the entire pipeline consistent.

```
icarus_stage1_producers.pandoraGausCryoW.HitFinderModuleLabel: "cluster3DCryoW"
icarus_stage1_producers.pandoraGausCryoE.HitFinderModuleLabel: "cluster3DCryoE"
```

---

### Fix 4: New FilterDecoder\_tool.cc (larrecodnn branch)

**File:** `larrecodnn/larrecodnn/NuGraph/Tools/FilterDecoder_tool.cc`

The cvmfs version of `FilterDecoder_tool.cc` (v10\_01\_10) used `idsmap[p][i]` --- the raw `art::Ptr` key --- as a direct positional index into `filtcol`, which is sized by the count of processed hits. Since hit keys index into the full collection (potentially much larger than the slice subset), this causes out-of-bounds writes and heap corruption.

Leonardo's `feature/nugraph-multislice` branch fixes this by building a sorted key list and using `std::find` + `std::distance` to convert keys to correct positional indices. It also introduces `art::Assns<FeatureVector<1>, recob::Hit>` for proper hit-score associations.

> This fix is the reason `lardataobj` and `larrecodnn` must be checked out from Leonardo's branch rather than cvmfs.

---

## 🔀 Pipeline: Before and After

### Before (initial v04p04 port -- crashes in FilterDecoder)

```
gaushit2dTPCEW/EE --> HitMerger --> combineHitsCryoE --> pandoraGausCryoE
gaushitPT2dTPCEW/EE --> cluster3DCryoE (SpacePoints)
                                             |
                                    nuslhitsCryoE (ICARUSNuSliceHitsProducer)
                                             |
                         NCCSlicesCryoE --> NGMultiSliceCryoE --> CRASH
                                           (key mismatch: combineHits != cluster3D)
```

### After (working) ✅

```
gaushit2dTPCEW/EE --> HitMerger --> combineHitsCryoE (runs but unused by Pandora)

gaushitPT2dTPCEW/EE --> cluster3DCryoE --> pandoraGausCryoE (HitFinderModuleLabel)
                                |                    |
                           (SpacePoints)       (slices + hit assns)
                                |                    |
                                +-----> nuslhitsCryoE (ICARUSNuSliceHitsProducer)
                                                     |
                                        NCCSlicesCryoE --> NGMultiSliceCryoE --> OK
                                                           (new FilterDecoder)
                                                                   |
                                                        ngfilteredhitsCryoE
                                                                   |
                                                     pandoraGausNuGraphRecoCryoE
```

Key principle: all hits feeding Pandora, NuGraph loader, and NuGraph decoder now come from a single consistent source --- `cluster3DCryoE/W`.

---

## 📊 NuGraph2 Hit Filter Statistics

Run on 5 MC overlay events (BNB RUN2, v10\_06\_00\_04p04 Stage 0 input). The filter score cut is 0.5. Numbers reported by `ICARUSFilteredNuSliceHitsProducer`.

| Event | Cryostat | Hits before filter | 1D model after | 1D % kept | 2D model after | 2D % kept |
|---|---|---|---|---|---|---|
| 1 | CryoE | 2165 | 1624 | 75.0% | 1716 | 79.3% |
| 1 | CryoW | 2961 | 2376 | 80.2% | 2522 | 85.2% |
| 2 | CryoE | 2793 | 2327 | 83.3% | 2458 | 88.0% |
| 2 | CryoW | 3629 | 3073 | 84.7% | 2168 | 59.7% |
| 3 | CryoE | 4804 | 3773 | 78.5% | 3858 | 80.3% |
| 3 | CryoW | 3989 | 3368 | 84.4% | 3435 | 86.1% |
| 4 | CryoE | 860 | 487 | 56.6% | 554 | 64.4% |
| 4 | CryoW | 505 | 224 | 44.4% | 352 | 69.7% |
| 5 | CryoE | 2983 | 1940 | 65.0% | 2530 | 84.8% |
| 5 | CryoW | 2568 | 2030 | 79.0% | 2177 | 84.8% |
| **Overall** | | **27257** | **21222** | **77.9%** | **21770** | **79.9%** |

> The 1D model column is shown for reference only --- it should not be used with 2D deconvolution inputs. The 2D model (`model_mpvmpr_1201.pt`) was trained on WireCell 2D deconvolution hits and is the correct model for v04p04.

> ⚠️ Event 2 CryoW shows an outlier (59.7% kept with the 2D model vs 84.7% with the 1D model). This may reflect genuine event-by-event variation and warrants further study on more events.

---

## 🧠 NuGraph2 Model

| | v10\_06\_00\_01p01 (1D) | v10\_06\_00\_04p04 (2D) |
|---|---|---|
| Model file | `model_mpvmpr_bnb_numu_cos.pt` | `model_mpvmpr_1201.pt` |
| Location | cvmfs `icarus_data/NuGraph/` | local (not yet on cvmfs) |
| Training sample | BNB $\nu_\mu$ + cosmics (1D hits) | MPVMPR (2D WireCell hits) |
| Semantic categories | MIP, HIP, Shower, Michel, Diffuse | MIP, HIP, Shower, Michel, Diffuse |

To use the 2D model, add its directory to `FW_SEARCH_PATH` before running:

```bash
export FW_SEARCH_PATH=/path/to/model/directory:$FW_SEARCH_PATH
```

The `modelFileName` parameter in `nugraph_icarus.fcl` is a FCL parameter --- no rebuild is needed to switch models.

> ⚠️ The 2D model should be submitted to `icarus_data` on cvmfs for production use.

---

## 📝 Notes on lardataobj and larrecodnn

These two packages are **not modified** in this port --- they are checked out from Leonardo Lena's `feature/nugraph-multislice` branch which contains the multi-slice NuGraph2 infrastructure needed for this port. The key change in `larrecodnn` is a fixed `FilterDecoder_tool.cc` that correctly maps `art::Ptr` keys to positional indices (Fix 4 above). The key change in `lardataobj` is updated class definitions for the multi-slice data products.

If these branches are merged into the LArSoft mainline in a future release, the explicit checkout step can be dropped.

---

## 📋 Pending PRs

The following changes should be submitted as pull requests to their upstream repositories once validated on more events:

| Change | Target repo | Priority |
|---|---|---|
| HitMerger `reserve()` fix | SBNSoftware/icaruscode | High -- pre-existing bug affecting all v04p04 users |
| NGMultiSliceCryoW FCL fix | SBNSoftware/icaruscode | High |
| Pandora `cluster3DCryoE` switch | SBNSoftware/icaruscode | Discuss with Riccardo -- has physics implications |
| NuGraph2 port (all 5 repos) | SBNSoftware/icaruscode etc. | After validation on more events |
| 2D model to cvmfs | icarus\_data | Coordinate with Riccardo |

---

## 🛠️ For Developers

See `for_developers/` for the full port guide and diagnostic notebooks. The structure is:

```
for_developers/
|
+-- docs/
|   +-- NuGraph2_Port_Guide_v01p01_v04p04.pdf    # detailed per-file changes,
|                                                  # common errors, checksums,
|                                                  # and fixes for this port
+-- notebooks/
    +-- diagnostic/
    |   +-- diagnose_caf_branches.ipynb           # CAF branch comparison
    |   |   +-- caf_branch_comparison_outputs/    # reference outputs
    |   +-- diagnose_stage1_labels.ipynb          # stage1 product label comparison
    |       +-- stage1_caf_propagation_outputs/   # reference outputs
    +-- patch_generation/
        +-- generate_package_diffs.ipynb          # generates .patch files for all repos
        +-- outputs/
            +-- fileListDiffs/                    # per-repo changed file lists
            +-- fullPatchDiffs/                   # full .patch files for git apply
            +-- diff_summary_all_packages.csv     # summary table
```

See `for_developers/README.md` for full details.

---

## 📬 Contact

**Sparshita Dey** -- ICARUS collaboration
GitHub: [@sparshitadey](https://github.com/sparshitadey)

With thanks to Giuseppe Cerati for debugging support, and to Leonardo Lena for the `feature/nugraph-multislice` larrecodnn/lardataobj branches.
