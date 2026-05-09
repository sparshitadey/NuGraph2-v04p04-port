# 🛠️ For Developers

This directory contains the port guide and diagnostic notebooks for the NuGraph2 v01p01 → v04p04 port. These tools are designed to be reused for future NuGraph2 ports across ICARUS and SBN production versions.

---

## 📁 Contents

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
    |   |   +-- caf_branch_comparison_outputs/    # reference outputs (CSVs + plots)
    |   +-- diagnose_stage1_labels.ipynb          # stage1 product label comparison
    |       +-- stage1_caf_propagation_outputs/   # reference outputs (CSVs + plots)
    +-- patch_generation/
        +-- generate_package_diffs.ipynb          # generates .patch files for all 7 repos
        +-- outputs/
            +-- fileListDiffs/                    # per-repo changed file lists
            +-- fullPatchDiffs/                   # full .patch files for git apply
            +-- diff_summary_all_packages.csv     # summary table
```

---

## 🔬 Diagnostic Notebooks

Run these **before touching any code** when starting a new port. They characterise what changed between versions so you know exactly where to focus your effort.

### 1. `diagnose_caf_branches.ipynb`

**When to use:** Before starting a new port.

**What it does:** Compares CAF branches between two icaruscode versions by pointing at one vanilla CAF file from each version. Produces a pairwise branch presence table showing which branches were added, removed, or renamed.

**What to look for:** Any branch in `only_A` (present in old version but not new) that has `ng_` or `NuGraph` in the name signals a CAF variable rename or removal that will need fixing in sbncode.

**Reference output for this port (v01p01 vs v04p04):**

| Comparison | Branches in A | Branches in B | Only in A | Only in B | Jaccard |
|---|---|---|---|---|---|
| v01p01 vs v04p04 | 2057 | 2106 | 0 | 49 | 0.977 |

Critical finding: `n_only_A = 0` --- no CAF branch was removed or renamed. All 49 new branches in v04p04 are pure additions. The NuGraph CAF filling code required no variable name changes.

---

### 2. `diagnose_stage1_labels.ipynb`

**When to use:** Before starting a new port.

**What it does:** Compares art product labels present in stage1 ROOT files between two icaruscode versions. Points at one vanilla stage1 file from each version and produces a label presence table.

**What to look for:** Hit collection labels that changed (e.g. `gaushit` -> `combineHitsCryoE`). Any NuGraph module or FCL that references the old hit label must be updated.

**Reference output for this port:**

| Product label | v01p01 | v04p04 | Significance |
|---|---|---|---|
| `pandoraGausCryoE/W` | present | present | Unchanged -- safe to reference |
| `cluster3DCryoE/W` | present | present | Unchanged -- safe to reference |
| `combineHitsCryoE/W` | absent | present | NEW -- 2D deco merge step |
| `ngfilteredhitsCryoE/W` | present | absent | Expected -- NuGraph not yet in v04p04 |

Critical finding: `combineHitsCryoE/W` replaces the v01p01 1D Gaussian hit source as the merged hit collection feeding into Pandora in v04p04.

> Note: the initial port used `combineHitsCryoE/W` as the NuGraph hit source, but this caused a key mismatch with `cluster3DCryoE/W` (used by the NuGraph decoder). The correct fix was to switch Pandora itself to `cluster3DCryoE/W` so that all hit sources are consistent. See Fix 3 in the main README.

---

## 📖 Port Guide

`docs/NuGraph2_Port_Guide_v01p01_v04p04.pdf` documents the complete v01p01 to v04p04 port in detail:

- **Part A:** Higher-level overview, dependency version table, diff scope per repo, CAF variable comparison, stage1 label comparison, recommended port sequence, testing checkpoints, pipeline data flow, branch naming conventions
- **Part B:** Full per-file changes for all 7 repositories, step-by-step instructions, common errors and fixes, checksum reference for sbnanaobj classes\_def.xml

### Using the port guide for a new port

1. Run both diagnostic notebooks to characterise what changed in the new version
2. Use the port guide as a template --- the structure of the port (which files to touch in which repos in which order) will be similar even if the specific changes differ
3. Pay particular attention to:
   - Section 3.1: dependency version check (run this first for the new version pair)
   - Section 4.5: the critical hit label fix (this is version-specific but the methodology applies)
   - Section B3: sbnanaobj checksums (these will be different for a new sbnanaobj tag)
   - Section B5: sbncode API changes (the NuGraph filling logic may have evolved further)

### Key lessons from this port

- **Check out lardataobj and larrecodnn from Leonardo's branch** (`feature/nugraph-multislice`) rather than cvmfs. The cvmfs version of `FilterDecoder_tool.cc` has a known indexing bug. If these branches have been merged upstream in a future release, this step can be skipped.
- **mrb uc does not reliably produce the correct build order.** Always edit `CMakeLists.txt` manually. The correct order is: larpandoracontent, lardataobj, larrecodnn, larpandora, sbnanaobj, icaruscode, sbncode.
- **There are three copies of every FCL file** in the dev area (srcs/, localProducts\*/, build\_slf7\*/). Any FCL edit must be applied to all three.
- **HitMerger has a pre-existing reallocation bug** in base v04p04. The fix (adding `reserve()`) should be PRed upstream and may be present in future releases.

---

## 📬 Contact

**Sparshita Dey** -- ICARUS collaboration
GitHub: [@sparshitadey](https://github.com/sparshitadey)
