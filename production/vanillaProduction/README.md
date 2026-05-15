# Grid Production Runbook — v10_06_00_04p04 Vanilla
**Author:** sdey2  
**Last edited:** 14th May 2026  
**Pipeline:** stage0 BNB ROOT → stage1 (2D deconv + Wiremod) → flat CAF  
**No NuGraph.** For NuGraph see `BNB_NuGraph_v10_06_00_04p04.xml`.
**LOCAL (gpvm).** See `/exp/icarus/app/users/sdey2/icaruscode-v10_06_00_04p04_production/vanillaProduction/miniProductionOutputRoot`
---

## 1. Environment setup

Run this every session before doing anything grid-related:

```bash
sh /exp/$(id -ng)/data/users/vito/podman/start_SL7dev_jsl.sh
source /cvmfs/icarus.opensciencegrid.org/products/icarus/setup_icarus.sh
setup icaruscode v10_06_00_04p04 -q e26:prof
cd /exp/icarus/app/users/sdey2/dev/dev_v10_06_00_04p04
source localProducts*/setup
mrbslp
```

Authenticate for grid (required for ifdh/dCache access):
```bash
httokensh -a htvaultprod.fnal.gov -i icarus -- /bin/bash
```
or if you just need a kerberos token:
```bash
kinit sdey2@FNAL.GOV
```

---

## 2. Local test run (GPVM, no grid)

Quick sanity check before submitting to the grid. Run 20 events through the full pipeline locally.
Output lives in:
```
/exp/icarus/app/users/sdey2/icaruscode-v10_06_00_04p04_production/vanillaProduction/miniProductionOutputRoot/
```

```bash
cd /exp/icarus/app/users/sdey2/icaruscode-v10_06_00_04p04_production/vanillaProduction/miniProductionOutputRoot
```

**Stage 1** (stage0 ROOT → stage1 ROOT):
```bash
lar -c stage1_run2_icarus_overlay_slimout.fcl -n 20 \
  -s /pnfs/sbn/data/sbn_fd/poms_production/mc/2025A_ICARUS_Overlays_BNB_RUN2/September/v10_06_00_04p04/stage0/00/02/<your_stage0_file>.root \
  --process-name redo -o stage1-vanilla-test.root --trace > stage1_vanilla.log 2>&1
```

Check it ran cleanly:
```bash
tail -50 stage1_vanilla.log
grep -i "error\|exception\|fatal" stage1_vanilla.log
```

**CAF** (stage1 ROOT → flat CAF):
```bash
export FW_SEARCH_PATH=$PWD:${FW_SEARCH_PATH}
lar -c cafmakerjob_icarus_detsim2d_systtools_and_fluxwgt_overlay.fcl -n -1 \
  -s stage1-vanilla-test.root --trace > caf_vanilla.log 2>&1
```

Check it ran cleanly:
```bash
tail -50 caf_vanilla.log
grep -i "error\|exception\|fatal" caf_vanilla.log
```

> Only move on to grid submission once both local stages complete without errors.

---

## 3. Make the tarball

The tarball packages your local dev area so grid worker nodes can set it up identically. You need to remake it any time you change FCLs or code.

```bash
cd /exp/icarus/app/users/sdey2/dev/dev_v10_06_00_04p04
make_tarball.sh /pnfs/icarus/resilient/users/sdey2/tarballs/icaruscode-workingarea-v10_06_00_04p04.tar.gz
```

Check it landed and is a sensible size (expect ~1–3 GB, never hundreds of GB):
```bash
ls -lh /pnfs/icarus/resilient/users/sdey2/tarballs/icaruscode-workingarea-v10_06_00_04p04.tar.gz
```

> **If it's huge (tens/hundreds of GB):** you likely have a `build_*` directory included.  
> Use the explicit exclude version instead:
> ```bash
> cd /exp/icarus/app/users/sdey2/dev/dev_v10_06_00_04p04
> tar --exclude='./build_*' --exclude='./.git' --exclude='./srcs/*/.git' \
>     -czf /pnfs/icarus/resilient/users/sdey2/tarballs/icaruscode-workingarea-v10_06_00_04p04.tar.gz .
> ```

---

## 4. XMLs

All XMLs live at:
```
/exp/icarus/app/users/sdey2/icaruscode-v10_06_00_04p04_production/vanillaProduction/
```

| File | Purpose |
|---|---|
| `v10_06_00_04p04_stage0-stage1.xml` | stage0 ROOT → stage1 ROOT (run first) |
| `v10_06_00_04p04_stage1-caf.xml` | stage1 ROOT → flat CAF (run after stage1 done) |
| `v10_06_00_04p04_stage0-caf_combined.xml` | Both stages in one XML (recommended) |

---

## 5a. Running with the combined XML (recommended)

The combined XML defines both stages. You run them sequentially — stage1 first, then fill in the stage1 output list and submit caf.

```bash
XML=/exp/icarus/app/users/sdey2/icaruscode-v10_06_00_04p04_production/vanillaProduction/v10_06_00_04p04_stage0-caf_combined.xml
```

**Submit stage1:**
```bash
project.py --xml $XML --stage stage1 --submit
```

**Monitor until complete:**
```bash
project.py --xml $XML --stage stage1 --status
```

**When done, check and harvest the output file list:**
```bash
project.py --xml $XML --stage stage1 --check
```
This writes a `files.list` into your stage1 bookdir:
```
/pnfs/icarus/scratch/users/sdey2/v10_06_00_04p04/standard/BNB_v10_06_00_04p04/stage1/test_1job/files.list
```

**Update the caf inputlist** in the XML — replace `FILL_IN_AFTER_STAGE1_COMPLETES` with that path, then:

```bash
project.py --xml $XML --stage caf --submit
```

**Monitor and check caf:**
```bash
project.py --xml $XML --stage caf --status
project.py --xml $XML --stage caf --check
```

---

## 5b. Running with the two separate XMLs

The two files are independent but feed into each other via the inputlist.

```bash
S1_XML=/exp/icarus/app/users/sdey2/icaruscode-v10_06_00_04p04_production/vanillaProduction/v10_06_00_04p04_stage0-stage1.xml
CAF_XML=/exp/icarus/app/users/sdey2/icaruscode-v10_06_00_04p04_production/vanillaProduction/v10_06_00_04p04_stage1-caf.xml
```

**Step 1 — Submit stage1:**
```bash
project.py --xml $S1_XML --stage stage1 --submit
```

**Step 2 — Wait, then check:**
```bash
project.py --xml $S1_XML --stage stage1 --status   # repeat until done
project.py --xml $S1_XML --stage stage1 --check    # harvests files.list
```

**Step 3 — Update the caf XML** with the path to the stage1 `files.list`, then submit:
```bash
project.py --xml $CAF_XML --stage caf --submit
```

**Step 4 — Monitor caf:**
```bash
project.py --xml $CAF_XML --stage caf --status
project.py --xml $CAF_XML --stage caf --check
```

> The two separate XMLs are useful for debugging stages independently. For routine production use the combined XML — one file to track, one place to update parameters.

---

## 6. Debugging failed jobs

```bash
# Find log tarballs in the logdir, then:
tar -tf log.tar                  # peek at contents
mkdir -p log_unpack && tar -xf log.tar -C log_unpack
cd log_unpack

# Search for errors in one pass (covers art, ifdh, Triton, file-not-found)
grep -rni "error\|exception\|failed\|cannot\|art::Exception" .

# Read logs directly
less larStage1.err
less larStage1.out

# Inspect the resolved FCL that actually ran on the node
cat Stage1.fcl
```

**Common exit codes:**

| Code | Cause | Fix |
|---|---|---|
| 90 | File not found (bad input path, wrong FCL name) | Check inputlist paths, FCL name spelling |
| 137 | OOM killed | Increase `<memory>` in XML |
| 1 | Generic art/lar failure | Read `.err` file, check `Stage1.fcl` |

---

## 7. Output locations

| Stage | Output |
|---|---|
| **Local** stage1 ROOT | `miniProductionOutputRoot/stage1-vanilla-test.root` |
| **Local** CAF ROOT | `miniProductionOutputRoot/` (flat CAF alongside stage1 output) |
| **Grid** stage1 ROOT | `/pnfs/icarus/scratch/users/sdey2/v10_06_00_04p04/standard/BNB_v10_06_00_04p04/stage1/` |
| **Grid** stage1 file list | `…/stage1/test_1job/files.list` (generated by `--check`) |
| **Grid** CAF ROOT | `/pnfs/icarus/scratch/users/sdey2/v10_06_00_04p04/standard/BNB_v10_06_00_04p04/caf/` |
| Logs | same directories as outputs |