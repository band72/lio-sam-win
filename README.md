# lio-sam-win

**Native Windows build of LIO-SAM Studio: ROS 2 Jazzy + LIO-SAM compiled with MSVC.**

A true Windows application with no Linux or WSL component. It packages ROS 2 Jazzy for Windows, GTSAM, PCL
and OpenCV built with Visual Studio, the LIO-SAM SLAM nodes, RViz2, and the LIO-SAM Studio GUI and
processing tools into one signed installer.

> **Status: planning / scaffolding.** Nothing is built yet. See [TODO.md](TODO.md).

This is **Route B** of the [Windows packaging analysis](https://github.com/band72/LIO-SAM-V2/blob/main/Enabling%20Windows%20Capable%20Solution.md):
estimated **~8–12 weeks** plus ongoing maintenance, **high risk** (mainly building GTSAM with MSVC).

## Approach

- `upstream/` is a pinned, unmodified submodule of [LIO-SAM-V2](https://github.com/band72/LIO-SAM-V2).
- Windows-specific changes live in `patches/` and are applied at build time. Each patch is a candidate to
  upstream into LIO-SAM-V2 behind platform checks, so `patches/` should shrink over time.
- Third-party C++ dependencies come from **conda-forge via pixi**, the same toolchain ROS 2 Jazzy uses on
  Windows, so ROS and LIO-SAM share one set of DLLs. Anything missing is built from source with a pinned
  manifest in `deps/`.

## Requirements

**Build machine:** Windows 10 22H2 or Windows 11 25H2 x64, **Visual Studio 2019 Build Tools (MSVC v142)**,
pixi, and the ROS 2 Jazzy Windows binary, set up per the ROS 2 Jazzy Windows install guide, under a short path
such as `C:\pixi_ws` with long paths enabled.

**End users:** Windows 11 25H2/26H2 or Windows 10 22H2 (while ESU lasts, to 2027-10-12), x64, 16 GB RAM
recommended, ~3 GB disk for the install.

> ROS 2 Jazzy's Windows guide states that only Windows 10 is supported upstream. Windows 11 support is this
> project's responsibility and must be covered by its own testing.

Full version matrix: see [TODO.md](TODO.md#target-platforms-and-versions-reviewed-2026-11).

## Repository layout

| Path | Purpose |
|---|---|
| `upstream/` | Git submodule → `LIO-SAM-V2` (`ros2` branch). **Not modified here.** |
| `deps/` | `pixi.toml`/`pixi.lock` and source-build scripts: GTSAM 4.2, PCL, OpenCV, Eigen, Boost, TBB, `pcl_conversions`, `cv_bridge` |
| `patches/` | Windows portability patches applied to `upstream/` at build time |
| `gui/` | Windows process-launch layer for the Tk GUI (replaces bash, `setsid`, `killpg`, `pkill`) |
| `installer/` | Installer definition (WiX or NSIS), DLL bundling, MSVC runtime |
| `ci/` | GitHub Actions workflows for `windows-2022` / `windows-2025` runners (with VS 2019 Build Tools installed) |
| `docs/` | Build guide and user guide |

## Getting the source

```bash
git clone --recurse-submodules https://github.com/band72/lio-sam-win.git
```

## Known porting work (summary)

| Area | Problem on Windows |
|---|---|
| GTSAM 4.2 + `gtsam_unstable` | MSVC build, Boost/TBB, DLL symbol export |
| `pcl_conversions`, `cv_bridge` | Likely not in the ROS 2 Windows archive; check RoboStack, else build from source |
| `src/mapOptmization.cpp` | `getenv("HOME")`; `system("rm -rf …")` and `mkdir -p` |
| PCL point types | MSVC alignment for custom point structs (`EIGEN_ALIGN16`) |
| GUI (`lio_sam_gui.py`, `lio_sam_record_gui.py`) | `source …setup.bash`, `preexec_fn=os.setsid`, `os.killpg`, `pkill` |
| Live recording | `ouster-ros` driver is Linux-only |

## License

MIT (see [LICENSE](LICENSE)), matching LIO-SAM-V2. The installer bundles third-party software under its own
licenses (ROS 2 Apache-2.0, GTSAM BSD, PCL BSD, OpenCV Apache-2.0, Qt LGPL, Boost BSL, TBB Apache-2.0,
Python PSF and packages).
