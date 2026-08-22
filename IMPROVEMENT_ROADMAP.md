# Physical Click-and-Drag Improvement Roadmap

## 1. Current Findings

`ImproveToMake.txt` requests a robust macOS-like one-finger physical click-and-drag feature for Magic Trackpad 2 on Windows.

The current pointer lock behavior is present in both input paths:

- USB path: `AmtPtpDeviceUsbUm/InputInterrupt.c`
- Bluetooth/HID filter path: `AmtPtpHidFilter/Input.c`

Both paths build a Precision Touchpad report, set `IsButtonClicked` from the Apple report button bit, then gate contact coordinate updates with `IgnoreButtonFinger`. When `PrevIsButtonClicked` and current `IsButtonClicked` are both true, the code enters the “lock the pointer” branch and reuses previous X/Y coordinates. That preserves click behavior but prevents the same held finger from moving the pointer during a physical drag.

Settings are registry-backed:

- USB reads values through `ReadSettingValue(...)`.
- Bluetooth reads values through `PtpFilterReadSettings(...)`.
- The control panel currently reads initial UI values from `HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\WUDF\Services\AmtPtpDeviceUsbUm\Parameters`.
- The control panel saves settings to both USB and Bluetooth service keys: `...\WUDF\Services\AmtPtpDeviceUsbUm\Parameters` and `SYSTEM\CurrentControlSet\Services\AmtPtpHidFilter\Parameters`.

No automated test project is visible in the repository.

## 2. Target Behavior

Add `AllowButtonHeldSingleFingerMotion`, default `1`.

When enabled, allow movement only for the narrow case:

- exactly one raw active contact;
- previous frame button down;
- current frame button down;
- current contact has `TipSwitch`;
- current contact ID matches the contact already tracked before or during the button-held frame;
- existing pressure, size, palm, and near-finger filters still pass.

When disabled, behavior must match the current implementation.

The implementation must treat this as an exception to the full pointer-lock decision, not as a simple replacement for the `IgnoreButtonFinger` condition. The current lock path also marks locked contacts by setting the high bit on `PrevPtpReportAux*.Id`; the exception must account for both `f->Id` and `UINT32_SET_MSB(f->Id)` states.

Before coding the `raw_n == 1` condition, confirm what `raw_n` means in each path. The one-finger condition must use the same pre-filter active-contact semantics that the existing pointer-lock path uses, not an assumed post-filter contact count.

## 3. Implementation Plan

### Phase 1: State and Settings

- Document the existing state transition first: normal contact tracking, initial `button up -> button down`, held `button down -> button down`, lock branch, and release/lift cleanup.
- Add `AllowButtonHeldSingleFingerMotion` to USB device context in `AmtPtpDeviceUsbUm/include/Device.h`.
- Add the same setting to Bluetooth driver context in `AmtPtpHidFilter/include/Driver.h`.
- Load the setting with default `1` beside `IgnoreButtonFinger`.
- Ensure reconnect/reinitialization starts from existing safe previous-contact state.
- Decide the install-time default strategy:
  - preferred: rely on code default `1` plus Control Panel persistence, matching the current INF style that does not preseed these tuning values;
  - alternative: add explicit `Parameters` values to both AMD64 and ARM64 INF files if release packaging requires visible defaults.

### Phase 2: Shared Decision Shape

- In both input paths, compute a local boolean such as `allowButtonHeldSingleFingerMotion`.
- Keep the condition local and allocation-free.
- Use existing contact IDs (`f->Id`) and previous auxiliary contacts (`PrevPtpReportAux1/2`) rather than adding a new tracking subsystem.
- If a tiny helper is practical without changing architecture, prefer a pure decision helper with equivalent USB/Bluetooth call sites. If that creates awkward coupling, duplicate the few-line condition deliberately and keep both copies identical.
- Build the exception around the full pointer-lock predicate:
  - current contact is not a new unrelated contact;
  - previous aux ID is either `f->Id` or `UINT32_SET_MSB(f->Id)`;
  - `raw_n == 1`;
  - `PrevIsButtonClicked && IsButtonClicked`;
  - `TipSwitch` is true;
  - pressure/size/palm/near-finger filters pass.
- Do not bypass filtering on the initial `button up -> button down` transition.

### Phase 3: USB Path

- Modify the pointer-lock decision in `AmtPtpDeviceUsbUm/InputInterrupt.c`.
- Permit coordinate update during the allowed held-button single-contact case.
- If the previous aux contact is currently MSB-marked, restore it to `f->Id` before updating X/Y so later frames continue normal tracking.
- Normalize only the previous auxiliary contact that corresponds to the current `f->Id`; do not clear MSB flags globally across unrelated aux contacts.
- Keep the pointer-lock branch for all other button-held cases.

### Phase 4: Bluetooth Path

- Apply equivalent logic in `AmtPtpHidFilter/Input.c`.
- Confirm default behavior and setting semantics match the USB path.

### Phase 5: Control Panel

- Add a checkbox to `AmtPtpControlPanel` labeled `Physical click and drag`.
- Read and write `AllowButtonHeldSingleFingerMotion`.
- Default checked when the registry value is absent.
- Preserve the current architecture: initial UI load reads from the USB WUDF key; save writes to both USB and Bluetooth service keys.
- Keep layout consistent with the existing `IgnoreButtonFinger` checkbox group.

### Phase 6: Documentation

- Add a concise README section describing:
  - the feature;
  - registry name and default;
  - USB and Bluetooth applicability;
  - difference from disabling `IgnoreButtonFinger`;
  - known limitation if contact ID changes during a held click.

### Phase 7: Diagnostics

- Add debug-only trace points for state transitions, not per-frame release logging.
- Useful events: exception activated, exception deactivated due to button release, finger lift, second finger, or contact ID mismatch.
- Do not add release-build high-frequency logs in the report hot path.

## 4. Verification Plan

Build targets:

- `msbuild AmtPtpDeviceUsbUm\MagicTrackpad2PtpDevice.vcxproj /p:Configuration=Release /p:Platform=x64`
- `msbuild AmtPtpHidFilter\AmtPtpHidFilter.vcxproj /p:Configuration=Release /p:Platform=x64`
- `msbuild AmtPtpDeviceUsbUm\MagicTrackpad2PtpDevice.vcxproj /p:Configuration=Release /p:Platform=ARM64`
- `msbuild AmtPtpHidFilter\AmtPtpHidFilter.vcxproj /p:Configuration=Release /p:Platform=ARM64`
- `msbuild AmtPtpControlPanel\AmtPtpControlPanel.csproj /p:Configuration=Release /p:Platform=AnyCPU`

State-table checks before hardware testing:

- `raw_n == 1`, previous button down, current button down, same contact ID, `TipSwitch == true`: coordinate update allowed.
- `raw_n == 1`, previous button down, current button down, same contact ID represented as `UINT32_SET_MSB(f->Id)`, `TipSwitch == true`: coordinate update allowed and only that aux contact is normalized.
- `raw_n == 1`, previous button up, current button down: existing behavior.
- `raw_n == 2`, button down: existing pointer-lock behavior.
- `raw_n == 1`, previous button down, current button down, same contact ID, `TipSwitch == false`: existing behavior.
- `raw_n == 1`, previous button down, current button up, same contact ID: exception disabled immediately.
- `raw_n == 1`, button released: existing behavior and no stale lock state.
- `raw_n == 1`, different contact ID while button held: existing/safe behavior.
- `raw_n == 1`, button held, then disconnect/reinitialize: no stale drag state survives.

Manual Windows checks:

- normal pointer movement;
- normal physical click;
- physical click-hold-move drag;
- desktop icon drag;
- window title-bar drag;
- window edge resize;
- text selection by click-drag;
- release click while finger remains touching;
- two-finger scroll;
- three/four-finger gestures;
- repeat over USB and Bluetooth.

## 5. Risks and Guardrails

- Do not set `IgnoreButtonFinger = 0` globally.
- Do not generate motion for multi-finger button-held frames.
- Prefer existing behavior if contact identity is ambiguous.
- Avoid high-frequency logging in the input hot path.
- Do not modify signing certificates, CAT files, or release signing workflow.
- A rebuilt driver will not retain Microsoft’s upstream signature.

## 6. Review Checklist Before Implementation

- The change does not globally disable `IgnoreButtonFinger`.
- USB and Bluetooth use equivalent conditions.
- `UINT32_SET_MSB(f->Id)` state is handled deliberately.
- The initial click transition is not broadened accidentally.
- Multi-finger button-held frames still use existing behavior.
- The Control Panel writes the setting to both service keys.
- INF default-value strategy is documented in the final implementation report.
- No signing artifacts, private certificates, generated packages, or unrelated formatting changes are included.
