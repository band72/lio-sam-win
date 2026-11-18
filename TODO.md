# TODO: lio-sam-win

Estimated effort: **~8–12 weeks**. Build platform: **Windows x64 with Visual Studio 2019 Build Tools**.

## Target platforms and versions (reviewed 2026-11)

| Component | Target | Notes |
|---|---|---|
| ROS 2 | **Jazzy Jalisco (LTS, EOL May 2029)**, official Windows binary | Matches LIO-SAM-V2 |
| Windows (officially supported by ROS 2 Jazzy) | **Windows 10** x64 | The Jazzy Windows install guide states "Only Windows 10 is supported". Windows 10 22H2 consumer ESU runs to 2027-10-12 |
| Windows (must also work) | **Windows 11 25H2 and 26H2** x64 | Not covered by upstream ROS testing; this project must test it. 24H2 Home/Pro left servicing 2026-10-13 |
| Compiler | **MSVC v142 (Visual Studio 2019 Build Tools)** | Must match the ROS 2 Jazzy Windows binaries; do not mix in v143-built C++ libraries without testing ABI |
| Dependency manager | **pixi + conda-forge**, as ROS 2 Jazzy for Windows uses | One set of DLLs for ROS and LIO-SAM's dependencies |
| Workspace path | Short root such as `C:\pixi_ws` / `C:\dev` | ROS on Windows hits the 260-character MAX_PATH limit; also enable `LongPathsEnabled` |
| Architecture | x64 only | |

## Phase 0: Project setup
- [ ] Pin `upstream/` to a tagged LIO-SAM-V2 release per version
- [ ] Build machine: Windows 10 22H2 or Windows 11 25H2, VS 2019 Build Tools (C++ workload), Git, pixi, ROS 2 Jazzy Windows binary zip, following the ROS 2 Jazzy Windows install guide
- [ ] Enable long paths (`LongPathsEnabled=1`, `git config --system core.longpaths true`) and keep all workspaces under a short root
- [ ] Turn on branch protection for `main` and require CI to pass

## Phase 1: Dependencies (`deps/`), 2–4 weeks, highest risk
- [ ] Add a project `pixi.toml` extending ROS 2's environment; take dependencies from **conda-forge** first (GTSAM, PCL, OpenCV, Eigen, Boost, TBB) so ROS and LIO-SAM share one ABI and one set of DLLs
- [ ] Check RoboStack (`robostack-jazzy` channel) for prebuilt `cv_bridge` and `pcl_conversions`; otherwise build `vision_opencv` and `perception_pcl` from source with colcon
- [ ] **GTSAM 4.2 with `gtsam_unstable`**: use the conda-forge build if it includes `gtsam_unstable` and matches LIO-SAM's API; otherwise build from source with v142, matching conda-forge's Boost/TBB/Eigen versions (fall back to a static build if DLL symbol export fails)
- [ ] Run GTSAM's unit tests on Windows before going further
- [ ] Use vcpkg only as a last resort, and never load two copies of Eigen, Boost or OpenCV in one process
- [ ] Lock versions (`pixi.lock`) and write one `deps/build.ps1` that reproduces everything from scratch

## Phase 2: LIO-SAM C++ portability (`patches/`), ~1 week
- [ ] `src/mapOptmization.cpp`: `getenv("HOME")` → `USERPROFILE` fallback
- [ ] `src/mapOptmization.cpp`: replace `system("rm -rf …")` and `mkdir -p` with `std::filesystem`
- [ ] Fix MSVC build errors: `M_PI` (`_USE_MATH_DEFINES`), MSVC OpenMP 2.0 loop forms, PCL custom point alignment (`EIGEN_ALIGN16`)
- [ ] `colcon build --merge-install` of `upstream/` with patches applied (merge-install keeps PATH entries short); all five nodes start
- [ ] Open PRs upstream for patches that are safe on Linux too

## Phase 3: GUI and processing tools (`gui/`), ~1 week
- [ ] Replace `source /opt/ros/jazzy/setup.bash && …` with Windows setup: `local_setup.bat` inside the pixi environment, or set PATH/AMENT_PREFIX_PATH in-process
- [ ] Replace `preexec_fn=os.setsid` and `os.killpg` with `CREATE_NEW_PROCESS_GROUP` plus `CTRL_BREAK_EVENT`, then `taskkill /T` as a fallback
- [ ] Replace `pkill -INT -f 'ros2 bag record'` with stopping the recorder's own process handle
- [ ] Replace `pcl_pcd2ply` with a Python PLY writer, or bundle the Windows PCL tool
- [ ] Install `rosbags`, `laspy[lazrs]`, `pyproj` (with PROJ data) and `scipy` into the pixi environment from conda-forge
- [ ] Windows default paths: `%USERPROFILE%\Documents\LIO-SAM\`

## Phase 4: Live recording
- [ ] Decide for v1: processing-only on Windows, or recording through the Ouster SDK (Windows-supported, records PCAP)
- [ ] If recording: add PCAP → ROS 2 bag conversion (~1–2 weeks)

## Phase 5: Packaging (`installer/`), 1–2 weeks
- [ ] Collect the runtime tree: pixi environment, ROS 2 install, LIO-SAM install, Qt (RViz2), PROJ data; install under a short path such as `C:\LIO-SAM\` or `%LOCALAPPDATA%\LIO-SAM`
- [ ] Bundle the VC++ 2015–2022 redistributable (x64)
- [ ] WiX v4+ or NSIS installer: Start-menu and desktop shortcuts, uninstaller, upgrades that keep user data
- [ ] Windows Firewall: localhost-only DDS (`ROS_AUTOMATIC_DISCOVERY_RANGE=LOCALHOST`) to avoid firewall prompts when no sensor is attached
- [ ] Code signing with **Azure Artifact Signing** (~$9.99/month) or an OV certificate; EV certificates no longer give instant SmartScreen trust
- [ ] `THIRD_PARTY_NOTICES.md` with all bundled licenses

## Phase 6: CI and testing (`ci/`), ~1 week
- [ ] GitHub Actions on **`windows-2022` or `windows-2025`** (`windows-2019` was retired 2025-06-30), installing VS 2019 Build Tools in the job; cache the pixi environment and dependency builds
- [ ] Regression test: process the 90 s Ouster sample bag and compare trajectory length with the Linux result (within 1%)
- [ ] Test the installer on clean VMs (no Visual Studio, no ROS): Windows 10 22H2, Windows 11 25H2, Windows 11 26H2
- [ ] Release checklist and upgrade notes in `docs/`

## Ongoing
- [ ] Rebuild and retest on each ROS 2 Jazzy Windows patch release and GTSAM update
- [ ] Before Windows 10 ESU ends (2027-10-12), confirm the next ROS 2 LTS's supported Windows and Visual Studio versions (REP 2000) and plan the move
