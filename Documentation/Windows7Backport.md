# Backporting ezEngine to Windows 7 (SP1)

This document describes the (best-effort) changes that make ezEngine build and run on
**Windows 7 SP1**. The work targets the engine *core* (Foundation, Core, RendererDX11) plus
the D3D11 samples; the editor and third-party plugins are left best-effort.

> The sandbox that produced these changes is Linux based, so nothing here was compiled or run on
> Windows. The changes are meant to be built and verified on a Windows machine with Visual Studio 2019.

## What was changed

| Area | File | Change |
| --- | --- | --- |
| Build option | `Code/BuildSystem/CMake/Platforms/Configure_Win.cmake` | New `EZ_TARGET_WINDOWS7` option. When enabled it lowers `_WIN32_WINNT`/`WINVER` to `0x0601` (NTDDI_WIN7) and defines `EZ_TARGET_WINDOWS7=1` for the code. |
| CMake preset | `CMakePresets.json` | New `vs2019x64-win7` configure preset (VS2019 / v142, `EZ_TARGET_WINDOWS7=ON`). |
| Generator | `GenerateWin7vs2019.bat` | Wrapper that runs `RunCMake.ps1 -Target vs2019x64-win7`. |
| Script | `RunCMake.ps1` | `vs2019x64-win7` added to the `-Target` validation set. |
| D3D11 device | `Code/Engine/RendererDX11/Device/Implementation/DeviceDX11.cpp` | `CreateAdapter()` now falls back from `IDXGIFactory6` (Windows 10 only) to `IDXGIFactory1::EnumAdapters1` when that interface is unavailable. |
| D3D11 feature level | `Code/Engine/RendererDX11/Device/Implementation/DeviceDX11.cpp` | The minimum required D3D feature level is relaxed from **11.1** to **11.0** on the Win7 target (stock Windows 7 only ships the D3D11.0 runtime). |
| Texture conversions | `Code/Engine/Texture/Image/Conversions/DXTexConversions.cpp` | `CreateHighPerformanceAdapter()` falls back from `IDXGIFactory6` to the first `IDXGIFactory1` adapter. |
| Mouse/keyboard input | `Code/Engine/Core/Platform/Win/InputDevice_Win.cpp` | `GetHardwareCursorSize()` no longer calls `GetDpiForWindow` / `GetSystemMetricsForDpi` directly (Windows 10 1607+); it routes through the already runtime-resolved `ezWindowsDpiUtils::GetWindowDpi()` and resolves `GetSystemMetricsForDpi` dynamically. |

## Why these are the only engine-owned blockers

Most Windows 8+ functionality in the engine is *already* resolved at runtime via
`GetProcAddress` (e.g. `SetThreadDescription`, `GetDpiForMonitor`, `GetDpiForWindow`,
`AdjustWindowRectExForDpi`, the DXGI debug interface) or is guarded behind
`_WIN32_WINNT >= _WIN32_WINNT_WIN8` (e.g. `CreateFile2` in DirectXTex, which switches to the
classic `CreateFile` path once `_WIN32_WINNT` is lowered). The D3D11 swap chain already uses the
Windows-7-compatible `DXGI_SWAP_EFFECT_DISCARD`. Condition variables and `CreateSemaphoreExW` are
Vista+ and therefore fine on Windows 7.

The three call sites above were the only ones that either *statically required* a Windows 8+ import
(`GetDpiForWindow`) or *hard-failed at runtime* on Windows 7 because they required the
`IDXGIFactory6` interface / a D3D feature level that stock Windows 7 does not provide.

## Toolchain requirements (build machine)

* **Visual Studio 2019** (the `v142` toolset). The default VS2022 (`v143`) runtime does **not**
  install or run on Windows 7, so VS2019 / v142 is required. VS2022 can be used *only* if the
  *"MSVC v142 - VS 2019 C++ x64/x86 build tools"* component is installed and forced via
  `-T v142`.
* A **Windows 10 SDK** (e.g. `10.0.19041.0` or newer) for the build. The headers (`dxgi1_6.h`,
  `d3d11_3.h`, ...) must be present; the target OS is selected purely through `_WIN32_WINNT`, not by
  using the deprecated Windows 7.1 SDK.
* CMake 3.22+ (the precompiled copy in `Data/Tools/Precompiled/cmake` is fine).
* The **Universal CRT** (`ucrtbase`) on the Windows 7 target machine (see below).

## How to build

```bat
REM From the repository root, on the Windows build machine:
GenerateWin7vs2019.bat
```

or equivalently:

```bat
powershell -NoProfile -ExecutionPolicy ByPass ./RunCMake.ps1 -Target vs2019x64-win7 -SolutionName AgentBuild -WorkspaceDir agent-build-win7
Data\Tools\Precompiled\cmake\bin\cmake --build Workspace\agent-build-win7 --config Debug -- -m -p:CL_MPCount=16
```

For a basic compilation smoke test, build the **Foundation** library first.

The `vs2019x64-win7` preset also sets `EZ_BUILD_FILTER=Win7` (see
`Code/BuildSystem/CMake/BuildFilters/Win7.BuildFilter`), which limits the solution to the engine core,
the D3D11 renderer and the main test executables — excluding the Editor (Qt) and the Vulkan backend so
the configure step does not pull in their extra SDK downloads.

### Continuous Integration

`.github/workflows/build-win7.yml` builds the engine on the `windows-2019` runner (v142 toolset) using
the `vs2019x64-win7` preset, then runs `FoundationTest` and `CoreTest` as a smoke test. It triggers on
push/PR to this branch and can also be started manually via `workflow_dispatch`.

## Runtime requirements (Windows 7 target machine)

* **Visual C++ 2019 redistributable (x64)** and the **Universal CRT**. On a clean Windows 7 SP1 the
  Universal CRT is not present; install it via the [KB2999226](https://support.microsoft.com/kb/2999226)
  update (or the convenience rollup / the VC++ 2019 redistributable, which bundles it).
* **(Optional) KB2670838 "Platform Update for Windows 7"** — installs the D3D11.1 runtime. Without it the
  engine runs at feature level **11.0** (what this backport accepts). With it, feature level **11.1** is
  available and used automatically.
* A Direct3D 11 capable GPU with drivers supporting at least feature level 11.0.

## Known limitations / things left best-effort

* The **editor** and the **Vulkan / WebGPU** backends are not part of the minimal scope and are
  unverified on Windows 7.
* Third-party libraries are compiled with the lowered `_WIN32_WINNT`. Most of them already resolve
  Windows 8+ APIs at runtime, but any dependency that *statically* imports a Windows 8+ symbol would
  fail to load on Windows 7; such a case would surface as a "procedure not found" loader error and would
  need the same runtime-resolution treatment.
* Feature level 11.0 (instead of 11.1) is accepted on the Win7 target. Shader/model code that genuinely
  depends on 11.1-only D3D features may need adjustments; install KB2670838 to regain 11.1.

## Verification checklist

1. `cmake --preset vs2019x64-win7` configures without errors and prints
   `Targeting Windows 7 SP1 (EZ_TARGET_WINDOWS7=ON): ...`.
2. The **Foundation** and **RendererDX11** libraries compile and link.
3. A D3D11 sample launches on a Windows 7 SP1 machine, initializes the device at feature level 11.0/11.1,
   and renders.
4. The same binaries still run correctly on Windows 10/11 (the runtime fallbacks are no-ops there because
   the Windows 8+ interfaces are present).
