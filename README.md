# Level Benchmark

Editor-only Unreal Engine plugin that benchmarks the currently loaded level: a synchronous static analysis pass (meshes, landscapes, lights) combined with PIE-based GPU timing, shader complexity, overdraw, and per-light cost passes. Results are viewable in-editor and exportable to CSV for tracking across runs.

> [!WARNING]
> **Plugin is made for lowpoly/stylized games and not for flim/rendering projects that don't mind 4K Textures and overlapping lights**
>
> Note, this project doesn't really do anything other than scanning or displaying stats, so your assets should be safe. But still, **BACK UP YOUR PROJECT**.

![Level Benchmark results window](Results.png)

*Benchmark ran on "White castle" by DmitriyDryzhak*

## Contents

- [Features](#features)
- [Requirements](#requirements)
- [Installation](#installation)
- [Usage](#usage)
- [Console Commands](#console-commands)
- [CSV Export](#csv-export)
- [Comparing Runs](#comparing-runs)
- [Flags Reference](#flags-reference)
- [Known Limitations](#known-limitations)
- [Support](#support)

## Features

- **Static pass** (no PIE required): per-component triangle count, LOD count, material count, max texture resolution, collision state, Nanite state, and instanced-mesh-aware effective triangle counts (`UInstancedStaticMeshComponent` / HISM).
- **Landscape scan**: component count, quad count, Nanite state per landscape proxy.
- **Light scan**: type, mobility, attenuation radius, shadow casting, pairwise overlap detection, and stationary shadow-cluster detection (shadow channel limit).
- **Material cost attribution**: top materials ranked by texture sample count and by usage among overdraw-prone blend modes, cross-referenced back into the mesh table.
- **Estimated texture memory**: real measured GPU resource size (`GetResourceSizeBytes`), deduplicated per unique texture across the level.
- **GPU pass timing** (PIE): per-pass GPU cost breakdown via the RHI breadcrumb profiler.
- **Shader complexity / quad overdraw** (PIE): sampled scene captures decoded against the engine's live color ramps, reported as percentage buckets with a worst-point hotspot.
- **Per-light GPU cost** (PIE, optional): isolates each shadow-casting non-static light's GPU cost by toggling it and diffing GPU time, with automatic camera placement near each light.
- **Run comparison**: diff the current run against a previously exported `Benchmark_Summary.csv`.
- **CSV export**: three files (meshes, lights, summary), auto-timestamped or user-chosen folder.
- Toolbar button, Tools menu entry, Window menu entry, and console commands for headless/automated runs.

## Requirements

- Unreal Engine 5.7.
- Windows (built and tested on Windows; other platforms untested).
- Editor-only — the plugin adds nothing to packaged builds.

## Installation

**Via Fab**

1. Purchase/acquire the plugin from its Fab listing.
2. Add it to your project through the Epic Games Launcher's Fab library, or download it directly to a project's `Plugins/` folder.
3. Open the project. If prompted to rebuild modules, accept.
4. Confirm the plugin is enabled: **Edit → Plugins → Editor**, search "Level Benchmark".

## Usage

### Opening the tool

Three equivalent entry points:

- Toolbar button (top-level editor toolbar)

![Toolbar Button](EP3.png)
- **Tools → Level Benchmark**

![Tools Button](EP1.png)
- **Window → Level Benchmark**

![Window Button](EP2.png)

### Settings

Opening the tool shows the settings window: a live light-count estimate for the loaded level, and checkboxes for which passes to run.

| Setting | Default | Notes |
|---|---|---|
| GPU timing | On | Requires a PIE session (Sequence A). |
| Shader complexity | On | Sampled scene capture, requires PIE. |
| Overdraw | On | Sampled scene capture, requires PIE. |
| Per-light GPU cost | Off | Runs one PIE session per qualifying light. Shows a time-estimate warning above 50 lights. |

![Settings window](Settings.png)

Click **Run Benchmark** to start. The static pass runs first and synchronously; its results are kept even if a later PIE pass is interrupted. PIE-based passes then run automatically — avoid manually closing PIE while a benchmark is in progress, as this is treated as an interruption (static results are still shown, with a banner and Retry option).

### Reading results

The results window opens automatically once a run finishes (or is interrupted).

- **Stat cards** — meshes scanned, average tri count, estimated texture memory, and (once measured) average frame/GPU time.
- **GPU Pass Breakdown** — per-pass GPU cost as sorted bars.
- **Shader Complexity / Overdraw** — percentage buckets with a hotspot callout and a "top suspect materials" list.
- **Meshes / Lights / Landscapes tables** — sortable, searchable, zebra-striped. Flagged rows are color-coded. Meshes have an **Open** button to jump straight to the asset editor.

![Mesh table with flags](Meshtable.png)

*mesh table screenshot*

![GPU breakdown and shader complexity](Gpupass.png)

*gpu breakdown and shader complexity breakdown*

## Console Commands

For headless or automated runs, bypassing all UI:

| Command | Effect |
|---|---|
| `LevelBenchmark.Run` | Static pass + Sequence A (GPU timing, shader complexity, overdraw), default settings. |
| `LevelBenchmark.RunWithLights` | Same as above, plus the per-light GPU cost pass (Sequence B). |
| `LevelBenchmark.ExportCsv` | Exports the most recent results to `Saved/BenchmarkExports/<timestamp>/`, bypassing the folder-picker dialog. |

## CSV Export

Click **Export CSV** in the results window, or use `LevelBenchmark.ExportCsv` for automation. Writes three files together into the chosen (or default) folder:

- `Benchmark_Meshes.csv`
- `Benchmark_Lights.csv`
- `Benchmark_Summary.csv`

The default folder is `Saved/BenchmarkExports/<timestamp>/`. All three files are required together for the comparison feature below to work — keep them in the same folder.

## Comparing Runs

Click **Compare with Previous...** and pick a previously exported `Benchmark_Summary.csv`. The tool loads that file (plus its sibling `Benchmark_Meshes.csv`/`Benchmark_Lights.csv` for flagged-row counts) and shows a delta table against the current run: mesh/light count, average tri count, average frame/GPU time, shader complexity (High+), average overdraw factor, and flagged mesh/light counts.

![Run comparison table](comparison.png)

This is an aggregate-only comparison — it does not match individual actors across runs, since actor identity isn't guaranteed stable across level edits between benchmarks.

## Flags Reference

**Meshes**

| Flag | Meaning |
|---|---|
| Missing collision | Component has collision disabled. |
| 4K+ textures with low tri count | Max texture resolution ≥ 4096 with fewer than 5000 triangles. |
| Duplicate material slots | The same material is assigned to more than one slot. |
| Instanced xN (effective M tris) | Instanced/HISM component; N instances, M = per-instance tris × N. |
| Uses high-complexity material (X) | References one of the level's top materials by texture sample count. |
| Uses overdraw-prone material (X) | References one of the level's top Translucent/Masked/Additive/Modulate/AlphaComposite materials by usage count. |

**Lights**

| Flag | Meaning |
|---|---|
| Movable+Shadow | Movable mobility with shadow casting enabled. |
| Large radius | Attenuation radius exceeds 25% of the level's largest bounding dimension. |
| Overlaps N other lights | 3 or more other lights' attenuation spheres intersect this one. |
| Exceeds 4-light shadow channel limit | Part of a cluster of 4+ overlapping stationary shadow-casting lights. |
| Confined viewpoint - cost may be unreliable | Per-light pass only: no clear line of sight was found to place the measurement camera; the cost figure is lower-confidence. |

**Landscapes**

| Flag | Meaning |
|---|---|
| Large landscape without Nanite | More than 16 components and Nanite is disabled. |

## Known Limitations

- Shader complexity and overdraw are sampled from 2–3 fixed camera points (PlayerStart actors, or level center as a fallback), not a full per-pixel heatmap.
- The GPU pass breakdown reads internal RHI breadcrumb data; verified against UE 5.7. Pass names may shift on other engine versions.
- Per-light GPU costs are not strictly additive — shadow channel packing changes when lights are toggled, so costs don't sum linearly to the level's total lighting cost.
- No hard cap on light count for the per-light pass, only a UI time-estimate warning above 50 lights.
- PIE settle time before capture is time-based (~1.5s), not frame-count-based.

## Support

Issues and questions: open an issue in this repository. The plugin itself is distributed and licensed through its Fab listing; usage is governed by the Fab EULA.
