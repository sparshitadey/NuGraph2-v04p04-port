# 🚧 Known issue: `sbncode` build fails on a fresh dev area — missing `product_deps` entries

If you've just set up a brand new `mrb newDev` area and `mrb i` fails immediately on `lardataobj`, this doc is for you — save yourself the debugging session I went through.

**Symptom:** A genuinely fresh `mrb newDev` area (no pre-existing `localProducts*` from an old build) fails during `mrb i` with:

```
CMake Error at .../cetmodules/.../CetOverrideFindPackage.cmake:177 (_find_package):
  By not providing "Findlardataobj.cmake" in CMAKE_MODULE_PATH this project
  has asked CMake to find a package configuration file provided by
  "lardataobj", but CMake did not find one.
Call Stack (most recent call first):
  sbncode/CMakeLists.txt:53 (find_package)
```

Fixing that one reveals the same error, one dependency at a time, for `larrecodnn`, `sbnanaobj`, `larpandora`, and `larpandoracontent`. CMake only reports the next missing dependency once the current one is resolved, so this surfaces incrementally rather than all at once.

## Root cause

`sbncode/CMakeLists.txt` calls `find_package()` directly on `lardataobj`, `larrecodnn`,
`sbnanaobj`, `larpandora`, and `larpandoracontent` — but none of these five products are
declared as dependencies in `sbncode/ups/product_deps` (neither in the product table nor
the qualifier table).

`mrb` uses `product_deps` — not the raw `find_package()` calls in `CMakeLists.txt` — to
decide build order and to populate `CMAKE_PREFIX_PATH` for products it's building from
local source. If a locally-checked-out product isn't declared in `product_deps`, `mrb`
has no way to know it needs to be built and installed before `sbncode`'s own configure
step runs.

**Why this doesn't show up in older/existing dev areas:** once a product has been built
and installed into `localProducts*/` at some point, CMake's `find_package()` just finds
the resulting `<product>Config.cmake` file sitting on disk — regardless of whether
`product_deps` ever declared the dependency. So areas with build history accumulated
over time work purely because the artifacts already exist, not because the dependency
declaration is correct. A genuinely fresh area has nothing built yet, so it's the first
to actually expose the gap.

## Fix

Only one file needs changing: `sbncode/ups/product_deps`. Nothing in `lardataobj`,
`larrecodnn`, `sbnanaobj`, `larpandora`, or `larpandoracontent` themselves needs to
change — they just need to be declared, not modified.

Add all five products to `sbncode/ups/product_deps`, in both tables:

**1. Product table** (insert before `cetmodules ... only_for_build`, before `end_product_list`):
```
lardataobj              v10_01_00       -
larrecodnn              v10_01_10       -
sbnanaobj               v10_00_05       -
larpandora              v10_00_19       -
larpandoracontent       v04_15_01       -
```
(adjust versions to match whatever tags/branches you've actually checked out)

**2. Qualifier table:** add one column per product, before the `notes` column
(anything appended after `notes` is not parsed correctly — see pitfall below).
Populate each row with that row's own qualifier value, e.g. for the `e26:prof` row,
the new columns should all read `e26:prof`. Do this for all four rows (`c14:debug`,
`c14:prof`, `e26:debug`, `e26:prof`).

## Pitfalls hit while fixing this

- **`sed` pattern matching on whitespace silently fails.** These files use tabs between
  columns, not spaces. A `sed` insert/match pattern written with literal spaces will
  silently do nothing — no error, just no change. Verify with
  `cat -A path/to/product_deps` (shows `^I` for tabs) before assuming an edit landed.
- **`notes` must stay the last column.** `mrb`'s parser treats `notes` as a reserved
  end-of-table marker. Appending new dependency columns after it produces a
  "dependency has no column in the qualifier table" error even though the column is
  visually present — it has to be inserted before `notes`.
- **`mrb uc` / `mrb updateDepsCM` will not fix this.** This is not a subdirectory-ordering
  problem in the top-level `CMakeLists.txt` — reordering `mrb_add_subdirectory()` calls
  has no effect on this error. The fix is entirely within `product_deps`.
- **Column-count mismatches between header and data rows fail silently**, with a
  different and more confusing downstream error rather than a clear "malformed table"
  message. Double check column counts match with
  `awk -F'\t' '{print NF, $0}' path/to/product_deps` after any edit.

## Verification

After applying the fix, `mrb i -j20` in a genuinely fresh `mrb newDev` area (freshly
cloned `icaruscode`, `sbncode`, `lardataobj`, `larrecodnn`, `larpandora`,
`larpandoracontent`, `sbnanaobj` — no pre-existing `localProducts*`) proceeds past CMake
configure and into actual compilation, confirming the dependency declarations are
correct.
