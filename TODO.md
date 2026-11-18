# TODO: lio-sam-win

Estimated effort: **~8–12 weeks**. Build platform: **Windows 10/11 x64 with Visual Studio 2019**.

## Phase 0: Project setup
- [ ] Create the public GitHub repo `band72/lio-sam-win` and push this scaffolding
- [ ] Set up the Windows build machine: VS 2019, ROS 2 Jazzy Windows archive, vcpkg, CMake, Python 3.12
- [ ] Pin `upstream/` to a tagged LIO-SAM-V2 release per version

## Phase 1: Dependencies (`deps/`), 2–4 weeks, highest risk
- [ ] **GTSAM 4.2** with MSVC, including `gtsam_unstable`: resolve Boost/TBB versions and DLL symbol export (fall back to a static build if the DLLs fail)
- [ ] Run GTSAM's unit tests on Windows before going further
- [ ] **PCL** via vcpkg (`pcl[core,io,filters,kdtree,registration]`); pin the vcpkg baseline
- [ ] **OpenCV** (vcpkg, or the one shipped with ROS 2 for Windows)
- [ ] Confirm whether `cv_bridge` and `pcl_conversions` are in the ROS 2 Jazzy Windows archive; if not, build `vision_opencv` and `perception_pcl` from source
- [ ] Write a `vcpkg.json` manifest and a single `deps/build.ps1` that reproduces everything from scratch

## Phase 2: LIO-SAM C++ portability (`patches/`), ~1 week
- [ ] `src/mapOptmization.cpp`: `getenv("HOME")` → `USERPROFILE` fallback
- [ ] `src/mapOptmization.cpp`: replace `system("rm -rf …")` and `mkdir -p` with `std::filesystem`
- [ ] Fix MSVC build errors: `M_PI` (`_USE_MATH_DEFINES`), OpenMP 2.0 loop forms, PCL custom point alignment
- [ ] `colcon build` of `upstream/` on Windows with patches applied; all five nodes start
- [ ] Open PRs upstream for patches that are safe on Linux too

## Phase 3: GUI and processing tools (`gui/`), ~1 week
- [ ] Replace `source /opt/ros/jazzy/setup.bash && …` with Windows environment setup (`local_setup.bat`, or set PATH/AMENT_PREFIX_PATH in-process)
- [ ] Replace `preexec_fn=os.setsid` and `os.killpg` with `CREATE_NEW_PROCESS_GROUP` plus `CTRL_BREAK_EVENT` or `taskkill /T`
- [ ] Replace `pkill -INT -f 'ros2 bag record'` with stopping the recorder's own process handle
- [ ] Replace `pcl_pcd2ply` with a Python PLY writer, or bundle the Windows PCL tool
- [ ] Verify `rosbags`, `laspy[lazrs]`, `pyproj` (with PROJ data) and `scipy` install on Windows Python 3.12
- [ ] Windows-friendly default paths (`%USERPROFILE%\Documents\LIO-SAM\`)

## Phase 4: Live recording
- [ ] Decide for v1: processing-only on Windows, or recording through the Ouster SDK (Windows-supported, records PCAP)
- [ ] If recording: add PCAP → ROS 2 bag conversion (~1–2 weeks)

## Phase 5: Packaging (`installer/`), 1–2 weeks
- [ ] Collect the runtime tree: ROS 2 install, LIO-SAM install, dependency DLLs, Qt (RViz2), Python, PROJ data
- [ ] Bundle the MSVC runtime (VC++ redistributable)
- [ ] WiX or NSIS installer: Start-menu and desktop shortcuts, uninstaller, upgrade path keeping user data
- [ ] Windows Firewall rules for ROS 2 DDS (or a localhost-only DDS configuration)
- [ ] Code signing certificate (~$200–500/year)
- [ ] `THIRD_PARTY_NOTICES.md` with all bundled licenses

## Phase 6: CI and testing (`ci/`), ~1 week
- [ ] GitHub Actions on `windows-2019`: cache the dependency builds, build LIO-SAM, build the installer
- [ ] Regression test: process the 90 s Ouster sample bag and compare trajectory length with the Linux result (within 1%)
- [ ] Test the installer on clean Windows 10 and Windows 11 VMs (no Visual Studio, no ROS)
- [ ] Release checklist and upgrade notes in `docs/`

## Ongoing
- [ ] Rebuild and retest on each ROS 2 Jazzy patch release and GTSAM update
