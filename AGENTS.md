# Repository Guidelines

## Project Structure & Module Organization

This repository contains a Windows Precision Touchpad driver stack for Magic Trackpad 2 and a WinForms control panel.

- `AmtPtpDeviceUsbUm/`: user-mode USB device driver project (`MagicTrackpad2PtpDevice.vcxproj`), with C sources at the module root and headers in `include/`.
- `AmtPtpHidFilter/`: HID filter driver project (`AmtPtpHidFilter.vcxproj`), with C sources and metadata headers.
- `AmtPtpControlPanel/`: C# WinForms control panel (`AmtPtpControlPanel.csproj`) with forms, resources, and manifest.
- `build/`: release packaging scripts and architecture-specific INF templates.
- `assets/`: README and documentation images.
- `.github/workflows/`: CI build workflow definitions.

## Build, Test, and Development Commands

Run commands from a Visual Studio 2022 developer environment with WDK support installed.

- `NuGet.exe restore AmtPtpHidFilter\packages.config -PackagesDirectory packages\`: restore driver NuGet packages.
- `msbuild AmtPtpDeviceUsbUm\MagicTrackpad2PtpDevice.vcxproj /p:Configuration=Release /p:Platform=x64`: build the USB driver for AMD64.
- `msbuild AmtPtpHidFilter\AmtPtpHidFilter.vcxproj /p:Configuration=Release /p:Platform=x64`: build the HID filter driver.
- `msbuild AmtPtpControlPanel\AmtPtpControlPanel.csproj /p:Configuration=Release /p:Platform=AnyCPU`: build the control panel.
- `build\make.bat`: create signed AMD64 and ARM64 output in `build\result`; requires certificate thumbprints and `inf2cat.exe`.
- `build\make_win10.bat`: package the Windows 10 AMD64 workaround from prebuilt signed binaries under `build\`.

## Coding Style & Naming Conventions

Follow the existing Visual Studio C/C# style. C driver code uses Windows driver naming patterns, PascalCase function names, and project-local headers in `include/`. Keep constants and hardware metadata near `include\Metadata` or `include\DeviceFamily`. C# UI code follows WinForms conventions: partial form classes, designer files, and PascalCase members. Avoid hand-editing `*.Designer.cs` or generated resources unless necessary.

## Testing Guidelines

There are no dedicated test projects. Verify changed projects with the relevant `msbuild` commands for `x64`; use `ARM64` too when touching shared driver code, INF packaging, or build scripts. For driver changes, validate package artifacts with `inf2cat.exe`.

## Commit & Pull Request Guidelines

Recent history uses short scoped subjects such as `build: Fix INF files`, `driver: Add support for Bluetooth`, and `readme: Add info about installation on Windows 10`. Keep commits focused and use a lowercase scope when practical.

Pull requests should include a concise description, affected hardware/Windows versions, build results, and manual validation. Include screenshots only for visible control panel changes. Link related issues for installation, compatibility, or device behavior fixes.

## Security & Release Notes

Do not commit private signing certificates, thumbprints tied to private keys, generated release packages, or local WDK/SDK paths. Keep signing and catalog generation in `build\` scripts, and avoid changing INF hardware IDs without documenting the device and Windows version tested.
