# Magic Trackpad 2 Precision Touchpad driver for Windows 11, signed by Microsoft, no more hacks required 🎉

## Fork notice

This repository is a personal fork of Vito Plantamura's Magic Trackpad 2 driver for Windows. I am grateful to Vito Plantamura, imbushuo, and all contributors whose reverse engineering, signing work, and maintenance made this project useful for Windows users.

The purpose of this fork is to experiment with and maintain a small behavior change for Magic Trackpad physical click-and-drag. In brief, this fork adds an `AllowButtonHeldSingleFingerMotion` setting that lets the same single finger continue moving the pointer while the physical button remains held, enabling macOS-like drag/drop, selection, moving, and resizing. The change is intentionally narrow and keeps the existing `IgnoreButtonFinger` behavior for multi-finger and unsafe contact transitions.

This fork respects the original project's GPLv2 license. Source changes are kept available under the same license, and upstream copyright, credit, and license notices should be preserved.

Because this fork modifies the driver binaries after the upstream Microsoft attestation process, the upstream Microsoft signature no longer applies to these builds. Current fork builds are provided as test-signed packages for development and personal testing, and they require trusting the included test certificate and enabling Windows test-signing mode.

Unofficial fork builds in this repository are not upstream or Microsoft-attested releases.

This is a fork of the excellent [imbushuo](https://github.com/imbushuo/mac-precision-touchpad) driver for the Magic Trackpad 2. **It supports Bluetooth**. Compared to imbushuo or to the official 2021 Apple driver, this project adds:

- support for USB-C Magic Trackpad 2
- battery level reading
- haptic feedback control
- various options for controlling pointer precision
- a control panel:

![Control Panel](https://raw.githubusercontent.com/vitoplantamura/MagicTrackpad2ForWindows/master/assets/ControlPanel.png)

## Physical click and drag

This fork supports macOS-like one-finger physical click-and-drag. When `AllowButtonHeldSingleFingerMotion` is enabled, the same finger that is holding the trackpad's physical click can continue moving the pointer, so normal Windows drag/drop, window movement, resizing and text selection work while the physical button remains held.

`AllowButtonHeldSingleFingerMotion` defaults to `1` and applies to both USB and Bluetooth paths. Setting it to `0` restores the previous behavior. This is a narrow exception to the existing `IgnoreButtonFinger` pointer-lock behavior; it does not globally disable `IgnoreButtonFinger` or change multi-finger gestures. If the device changes the contact ID unexpectedly while the physical click is held, the driver prefers the existing safe behavior, so a drag may stop rather than risk a cursor jump.

The control panel reads the USB WUDF registry settings first and falls back to the Bluetooth filter settings when the USB key is not present. Saving mirrors the same settings to both driver paths. Reconnect the trackpad or restart the driver after changing these options so the running driver reloads them.

The previous version of this project used a hack to install itself in the DriverStore and couldn't support Bluetooth. At the beginning of this year, I decided to purchase an EV certificate to properly sign the driver: I paid 485 euros for it, including taxes that I have no way of recovering as an individual (btw, only organizations can request an EV certificate). I was tired of seeing people resorting to the wildest hacks to get the MT2 to work via Bluetooth 😀 (you can get a glimpse of this in the issues of this repo). **Windows drivers signing requirements and costs are unfair to open-source developers**.

## Installation on Windows 11

0) Uninstall any previous versions of this driver, imbushuo or `official 2021 Apple driver`. Personally I use [DriverStore Explorer](https://github.com/lostindark/DriverStoreExplorer) for that, alternatively you can use Windows Device Manager. Also, **it's especially important to uninstall `Magic Utilities` and `Trackpad++`** before continuing with the installation!

1) Download the zip file of this project from the [Releases](https://github.com/vitoplantamura/MagicTrackpad2ForWindows/releases) of this repo and unzip it.

2) Select your architecture: AMD64 or ARM64. Right-click on the INF file and click "Install".

### Note about Windows 10

Windows 10 AMD64 is supported through [this workaround](https://github.com/vitoplantamura/MagicTrackpad2ForWindows/issues/25#issuecomment-4084774884).

Windows 10 ARM64 is not supported.

## Credits

- [This excellent PR](https://github.com/imbushuo/mac-precision-touchpad/pull/533) of [1Revenger1](https://github.com/1Revenger1) to the imbushuo repo, which fixes the "near field fingers" problem, cleans up the code, and removes the QueryPerformanceCounter call in the interrupt function.
- The haptic feedback control messages sent by the driver to the MT2 in this project are based on the excellent reverse engineering work of [dos1](https://github.com/dos1) ([here](https://github.com/mwyborski/Linux-Magic-Trackpad-2-Driver/issues/28#issuecomment-451625504)).
- My long-time friends at [Landlogic IT](https://landlogic.it/), who took care of the grueling process of gaining access to Microsoft's Hardware Dashboard and who take care of signing the driver packages for me.
- @ordens, Taylor Sharp, @Wikiwix, @nagromc, @danspel, 乔​泽昱, Purasu Oy, Patrick Adler for their contribution to the purchase of the EV certificate ([more info here](https://github.com/vitoplantamura/MagicTrackpad2ForWindows/issues/31)).
