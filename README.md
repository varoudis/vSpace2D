# vSpace2D — macOS binary release

High-Performance experimental command-line tool for 2D segment-graph spatial analysis developed by Tasos Varoudis. Operates directly on a segment-map CSV for angular / metric / topological integration, betweenness/choice, and reach.

- **Version:** `0.95ang`
- **Platform:** macOS on **Apple Silicon (arm64)** only — will not run on Intel Macs.
- **Self-contained:** the OpenMP runtime is bundled; no Homebrew or other install is required.

> **Citation requirement.** This is experimental academic software. **Always
> reference the tool *and* the specific version used** (`0.95ang`) in any
> publication or report that includes results produced with it. Behavior changes
> between releases, so the version number matters. Free for non-profit academic
> research; provided **AS IS** with no warranty of any kind. Source code is not
> included in this release and will be release once I have a bit of time to make it
> more presentable... This was coded before the age of LLMs so the code is messy :)

It was used internally at the Space Syntax Lab at UCL since around 2015 and can run quite fast on very big multi-core systems (more than 256 cores). If you need a linux build be let me know and I'll release one.
---

## What's in this folder

| File | Purpose |
|---|---|
| `vSpace2D_macos_arm64` | Command-line build (standard). |
| `vSpace2D_macos_arm64_tui` | Interactive terminal-UI build (guided flag wizard + live progress). Run with `--tui`. |
| `libomp.dylib` | Bundled OpenMP runtime. **Keep it next to the binaries** — they load it from their own folder. |
| `README.md` | This file. |

Keep the two binaries and `libomp.dylib` together in the same directory. The
binaries locate `libomp.dylib` relative to themselves (`@loader_path`), so moving
a binary out on its own will stop it launching.

---

## First run (Gatekeeper)

The binaries are ad-hoc signed, not notarized. If you downloaded this folder
(browser, AirDrop, zip), macOS quarantines it and will refuse to launch it the
first time. Clear the quarantine flag once, from inside this folder:

```sh
xattr -dr com.apple.quarantine .
chmod +x vSpace2D_macos_arm64 vSpace2D_macos_arm64_tui
```

Alternatively, right-click the binary in Finder → **Open** the first time and
confirm the prompt.

---

## Quick start

Run angular integration + betweenness on your segment map:

```sh
./vSpace2D_macos_arm64 \
  -ang \
  -f city.csv \
  -r radii.txt \
  -o Analysis_Output
```

Outputs:

- `Analysis_Output.csv` — segment table with `vS_*` analysis columns.
- `Analysis_Output_WKT.csv` — same data with WKT geometry.

If `-r` is omitted, the default radii are `500, 1000, 2000, 4000, 5000` (meters).

Interactive (guided) mode:

```sh
./vSpace2D_macos_arm64_tui --tui
```

---

## Input format

Segment-map CSV with a header row. Required columns:

| Column | Meaning |
|---|---|
| `Ref` | Segment ID (int). Used as the `nRef` in all outputs and sidecar files. |
| `x1,y1,x2,y2` | Endpoint coordinates (projected, metric). |
| `Connectivity` | Topological connectivity at junctions. |
| `Segment Length` | Geometric length of the segment. |

An `Angular Connectivity` column is read if present. Extra columns are ignored.
The radii file (`-r`) is plain text, one radius value per line.

---

## CLI reference

### Required

| Flag | Description |
|---|---|
| `-f FILE` | Input segment-map CSV. |
| `-o NAME` | Output filename prefix. All CSVs and sidecar files share this prefix. |

### Shortest-path mode (pick one; default `-ang`)

| Flag | Dijkstra edge cost |
|---|---|
| `-ang` | Angular weight (turn cost). |
| `-metric` | Metric distance (segment length). |
| `-topo` | Topological (1 per edge). |

### Radii

| Flag | Description |
|---|---|
| `-r FILE` | File listing radius values, one per line. Defaults to `500,1000,2000,4000,5000`. |
| `-radius-mode MODE` | How each path is *measured* against the radius values: `metric\|topo\|angular` (short forms `m`, `t`, `a`). **Default `metric`** — radii are meters unless this flag is supplied. |

Shortest-path mode and radius mode are independent. `-ang -radius-mode metric`
means "angular Dijkstra, measured against metric radii". Within a single run all
radii share one radius mode; mixing radius modes in one run is not supported.

Output columns encode the radius mode as a single-character suffix: `m` (metric),
`t` (topological), `a` (angular). So `vS_Node_Count_R500m` is metric R=500,
`vS_Node_Count_R3t` is topological R=3 steps, and fractional radii keep a lossless
label such as `vS_Node_Count_R1.2a`. Duplicate radius values are rejected.

The legacy special-weight flags (`-sp`, `-spf` / `--spfactor`,
`-radius-mode special`) are disabled and return an error.

### Analysis type

| Flag | Description |
|---|---|
| *(default)* | `IntAndBC` — Integration + Betweenness/Choice in one Dijkstra pass. |
| `-reach` | Reach-only mode. No BC, no integration — much faster on huge maps. Emits per-radius scalar columns plus CSR sidecar files of the nRefs inside each reach. |

### Angular cost modifiers

| Flag | Description |
|---|---|
| `-ab N` / `--ang-bins N` | Quantize angular turn cost into N bins (`0` = disabled; minimum enabled value is `2`). |
| `-angle-precision P` | Snap raw turns to the nearest multiple of `P` degrees (0 = disabled). |
| `-angle-threshold T` | Treat raw turns below `T` degrees as straight (0 = disabled). |
| `-noanti` | Disable the anti-U-turn penalty in angular Dijkstra. |

### Other

| Flag | Description |
|---|---|
| `-no_rn` | Skip the global/N (no-radius) outputs. |
| `--zero-dead-ends` | Zero BC for non bi-connected lines. |
| `-u FILE` / `--unlink FILE` | Unlinks file. |

---

## Outputs

### Main CSV (`<prefix>.csv`, `<prefix>_WKT.csv`)

One row per segment, all input columns plus `vS_*` analysis columns. Suffixes:

- `_N` — global / no-radius variant (omitted when `-no_rn`).
- `_R<value><mode_tag>` — per-radius variant, e.g. `_R500m`, `_R3t`.

#### `IntAndBC` columns (default)

| Column prefix | Meaning |
|---|---|
| `Connectivity` | Out-degree of the segment in the graph. |
| `vS_Node_Count_*` | Source-inclusive node count `N` (reachable targets + the origin). |
| `vS_Ang_Total_Depth_*` | Sum of shortest-path distances (shortest-path mode units). |
| `vS_Ang_Mean_Depth_*` | Mean depth `TD / (N - 1)`. |
| `vS_Closeness_*` | Closeness centrality (1 / total depth). |
| `vS_exp_AngIntHH_*` | Hillier-style integration value. |
| `vS_dX_AngInt_*` | Depth-based integration (`N² / TD`). |
| `vS_AngInt_Turner_*` | Turner-normalized integration (`(N - 1) / (1 + TD)`). |
| `vS_AngInt_NAIN_*` | NAIN integration (`N^1.2 / (TD + 1)`). |
| `vS_AngInt_Hillier_*` | Hillier integration (`N² / (TD + 1)`). |
| `vS_Ang_BC_*` | Betweenness / choice as the full directional sum. |
| `vS_BC_Turner_*` | Directional BC normalized by `(N - 1)(N - 2)` when `N > 2`. |
| `vS_BC_NACH_*` | NACH choice (`log10(BC + 1) / log10(2 + TD)`). |

#### `Reach` columns (`-reach`)

| Column | Meaning |
|---|---|
| `vS_Reach_Count_R*` | Number of *other* segments reachable from the source within R (excludes the source). |
| `vS_Reach_Length_R*` | Sum of `Segment Length` over reachable segments within R. |

### Reach sidecar files (`-reach`)

Per radius value, two int32 binary memmaps in standard CSR layout:

```
<prefix>_R<value><mode_tag>_ReachOff.mmap   shape (V + 1,)
<prefix>_R<value><mode_tag>_ReachIDs.mmap   shape (offsets[V],)
```

For source segment `i`, the nRefs inside its reach are `ids[off[i]:off[i+1]]`.

Reader (Python):

```python
import numpy as np
off = np.fromfile("Analysis_Output_R500m_ReachOff.mmap", dtype=np.int32)
ids = np.fromfile("Analysis_Output_R500m_ReachIDs.mmap", dtype=np.int32)
reach_of_segment_i = ids[off[i]:off[i+1]]
```

---

## Examples

```sh
# Angular integration, default radii
./vSpace2D_macos_arm64 -ang -f city.csv -o out

# Metric Dijkstra, custom radii (still measured as meters)
./vSpace2D_macos_arm64 -metric -r radii.txt -f city.csv -o out

# Angular Dijkstra constrained by topological step radii
./vSpace2D_macos_arm64 -ang -radius-mode topo -r topo_steps.txt -f city.csv -o out

# Reach-only run (no BC cost) for accessibility analysis
./vSpace2D_macos_arm64 -ang -reach -r radii.txt -f city.csv -o out

# Angular cost rounded to 5° increments, small turns ignored
./vSpace2D_macos_arm64 -ang -angle-precision 5 -angle-threshold 10 -f city.csv -o out
```

---

## Third-party notice

`libomp.dylib` is the LLVM OpenMP runtime, redistributed here under the Apache
License v2.0 with the LLVM exception. See <https://llvm.org> for details. All
other code in the vSpace2D binaries is © Tasos Varoudis and is not open source.
