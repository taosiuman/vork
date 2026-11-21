# PR #21 evidence - std/path verification on RTX 5090

Collected 2026-11-21 with the official vork 0.6.1 release binary
(commit e4d7c98, API v2). All renders use `--profile std/path`.

## Environment

- NVIDIA GeForce RTX 5090, 32607 MiB, WDDM
- Driver 621.24 (KMD 621.24, CUDA UMD 13.5), nvgpucomp64.dll 32.0.16.2124
- Vulkan instance 1.4.357
- Windows build 26300.9974 (DisplayVersion 26H2; registry ProductName still says "Windows 10 Pro")
- Slang 2026.21, OIDN GPU denoiser

## Runs

| Scene | Exit | Wall | Shader jobs | Driver pipelines | `unidirectional path` driver compile |
|---|---:|---:|---:|---:|---:|
| starter (default cache, 800x600) | 0 | ~1 s | 0/0 | 10/10 | cache hit |
| starter (fresh cache, 800x600) | 0 | 1.15 s | 0/0 | 10/10 | cache hit |
| examples/first-scene.json | 0 | 2.03 s | 0/0 | 10/10 | 1.40 s |
| examples/product-orbit.json | 0 | 2.23 s | 0/0 | 10/10 | 1.40 s |
| examples/programs.json | 0 | 18.27 s | 12/12 | 16/16 | 1.55 s |
| examples/black-hole/scene.json --at 6 | 0 | 28.38 s | 35/35 | 40/40 | 5.78 s |
| examples/fog-interior.json | 0 | 63.08 s | 8/8 | 11/11 | 8.80 s |

`deferred_variants = 0` in every run; every variant compiled inline.
No new Windows Error Reporting entry appeared during these runs.

## Windows Error Reporting (only vork.exe event in last 24 h)

From the initial `vork setup` at 2026-11-21 01:06:55:

    P1: vork.exe   P2: 0.0.0.0        P3: 6afd062d
    P4: nvgpucomp64.dll   P5: 32.0.16.2124   P6: 6aef2e6c
    P7: c0000005   P8: 0000000000a8c6e4

## Files

- `path-t1.png` / `path-t1b.png` - starter scene, default / fresh cache
- `t2-*.png` ... `t6-*.png` - rendered outputs (320x180)
- `*.log` - full stdout/stderr of each render
- `*.png.timings.json` - per-pipeline / per-shader timings, pipeline cache stats
- `*.png.meta.json` - render metadata
- `blackhole-generate.log` - asset generation for the black-hole scene
- `reply-draft.md` - the text posted to PR #21

## Note

`examples/blend.json` does not exist in the 0.6.1 export tree or the release
archive, so the shipped scenes documented as requiring `std/path` were used.
