Thanks — understood on the export/release path, and on crediting. Happy to leave the PR as it is.

## Short answer: path tracing works fine on this machine

`std/previs` is not the only thing that runs here. I exercised `std/path` across six scenes, including the shipped example whose README says `std/path` is required, and every one compiled and rendered. No new crash, and no new Windows Error Reporting entry.

One note first: **`examples/blend.json` does not exist** in the 0.6.1 export tree or in the release archive (I checked both). So I substituted the shipped scenes the docs say need `std/path` (`examples/black-hole`) plus the interior/program/generic examples. If `blend.json` is main-tree only, send it over and I'll run it exactly as written.

## Environment

- NVIDIA GeForce RTX 5090, 32607 MiB, WDDM
- Driver 621.24 (KMD 621.24, CUDA UMD 13.5)
- Vulkan instance 1.4.357
- Windows build 26300.9974 (DisplayVersion 26H2; the registry `ProductName` string still says "Windows 10 Pro")
- vork 0.6.1 official release, commit `e4d7c98` (API v2)
- Slang 2026.21, OIDN GPU denoiser

## Runs (official 0.6.1 binary, `--profile std/path`, 320x180 unless noted)

| Scene | Exit | Wall | Shader jobs | Driver pipelines | `unidirectional path` driver compile | Output |
|---|---:|---:|---:|---:|---:|---|
| `vork init` starter (800x600) | 0 | 1.15 s | 0/0 | 10/10 | cache hit | valid |
| `examples/first-scene.json` | 0 | 2.03 s | 0/0 | 10/10 | 1.40 s | valid |
| `examples/product-orbit.json` | 0 | 2.23 s | 0/0 | 10/10 | 1.40 s | valid |
| `examples/programs.json` | 0 | 18.27 s | 12/12 | 16/16 | 1.55 s | valid |
| `examples/black-hole/scene.json --at 6` | 0 | 28.38 s | 35/35 | 40/40 | 5.78 s | valid |
| `examples/fog-interior.json` | 0 | 63.08 s | 8/8 | 11/11 | **8.80 s** | valid |

Every output is a real picture — the black-hole still shows the shadow, the lensed disk and the photon ring as expected, and the fog interior shows direct plus bounced fog lighting. `deferred_variants` is 0 in every run, so nothing was deferred; each variant compiled inline.

## Where the crash actually sits

The setup warmup calls `compute_entry(..., "unidirectional path")` from the prebuilt `PATH_ENGINES` module (`passes/path.rs:438`), and that is where the driver dies — on the 4th prebuilt family (`path_engine_1_1`, adaptive). A render reaches `unidirectional path` through `install_hit_selected()` (`passes/path.rs:496`), i.e. a scene-specialised variant compiled on demand. All of those succeed, including 8.8 s of driver time for the fog interior.

So the fault looks specific to the prebuilt *variant* compiled during warmup, not to path transport in general and not to the "unidirectional path" kernel as a concept. That is consistent with your plan to shrink those shaders — but it also suggests the prebuilt families and the on-demand variants are different enough that the prebuilt one is the trigger.

## Windows Error Reporting

Only one `vork.exe` entry in the last 24 h, from the initial `vork setup` at 2026-11-21 01:06:55:

```
P1: vork.exe   P2: 0.0.0.0        P3: 6afd062d
P4: nvgpucomp64.dll   P5: 32.0.16.2124   P6: 6aef2e6c
P7: c0000005   P8: 0000000000a8c6e4
```

Nothing appeared during the six `std/path` runs above. Note that `setup` did publish 3 of the 4 families before it died, and those cache entries survive and are reused.

## Compile cost, if it helps the shrink work

Driver time is dominated by the path pipeline and scales with scene complexity:

- `unidirectional path`: 1.40 s (simple) → 5.78 s (black hole) → 8.80 s (fog interior)
- Slang time is separate and larger for authored content: 51.7 s total for fog-interior, 17.2 s black hole, 13.7 s programs
- Pipeline cache reports `status=accepted`, saves succeed, and the blob grew from ~1.0 MB to ~4.1 MB across the runs

Per-pipeline and per-shader timings are in the `.timings.json` sidecars; I can paste the raw tables if you want them.

Happy to run a specific scene, a reduced-shader build, or `blend.json` from main if you can share it.
