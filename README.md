# WhateverXe (Alder Lake-P)

Lilu plugin providing graphics acceleration and internal display (eDP) support for 12th-Gen Intel Iris Xe Graphics (`8086:46A6`, Alder Lake-P) on macOS Tahoe (26.x) and macOS Sequoia (15.x).

Tested on a **Dell Latitude 7430** (Intel Core i7-1270P) with Metal 2 acceleration, native backlight control, display sleep/wake, and hardware reset recovery.

---

## Overview & Patches

Official WhateverXe (v0.8.0–v0.8.2) fails on 12th-Gen Alder Lake-P laptops because firmware hands off the internal eDP panel in port mode 0, which upstream rejects. As a result, `AppleIntelFramebuffer@0` fails to bind, WindowServer falls back to `.Display_boot` (`IONDRVFramebuffer`), and `IntelAccelerator` never loads (reporting 7 MB VRAM and no Metal).

This repository incorporates the following fixes:

1. **eDP Port Mode 0 Hand-off (`dca0ff6`):** Admits the internal panel during firmware hand-off in native port mode 0, allowing `AppleIntelFramebuffer@0` to claim the display.
2. **Single-Output CDCLK Routing (`490e06e`):** Computes and sets the 1305.6 MHz core display clock path when only the internal eDP pipe is connected.
3. **Flexible Aperture Bounds (`8321cc9`):** Accepts stolen memory apertures that do not end on a strict 4KB page boundary.
4. **Display Engine (DE) Sleep/Wake Management (`0859ce7`–`5ec4073`):** Parks DE master interrupts while the display pipe is powered down and re-arms them on wake, preventing blitter ring hangs.
5. **Selective GVA Unpublishing (`-wexgvadrop`):** Selectively unpublishes unstable Tiger Lake VA driver keys on Alder Lake-P, preventing hardware codec (`VCS0`) ring faults and UI freezes while keeping HEVC GPU decode active.
6. **Hardware Reset & RCS Recovery (`-wex-reset-restore`):** Intercepts GPU hardware engine resets (`GDRST`), preserves and restores Render Command Streamer (RCS) registers (execlists, status page, chicken bits, whitelist, and MOCS/L3 cache tables), and sweeps dropped event stamps, recovering the GPU within seconds without system lockup.
7. **Single Context Enforcement (`-wex-no-multicontext`):** Directs compute tasks through the primary render engine to prevent Alder Lake multi-context deadlocks.
8. RPCS full-slice, GSYNC preemption fix for improving ui smoothness and heavy task performance

---

## Hardware Profile

| Component | Specification |
| :--- | :--- |
| Model | Dell Latitude 7430 |
| Processor | Intel Core i7-1270P (Alder Lake-P, 12C / 16T) |
| Integrated Graphics | Intel Iris Xe Graphics (`8086:46A6`, Rev `0x0C`, Subsystem `1028:0B09`) |
| Display Panel | 14.0" 1920x1080 60Hz eDP |
| Operating System | macOS Tahoe 26.7.3 (`25G342`) / macOS Sequoia 15.x |
| SMBIOS | `MacBookPro16,2` |
| VRAM | 1536 MB Dynamic VRAM |
| Metal Support | Metal 2 (`AppleIntelTGLGraphicsMTLDriver`) |
| Driver Pipeline | `IGPU@2` -> `AppleIntelFramebuffer@0` -> `display0` -> `AppleBacklightDisplay` |

---

## Status

### Working
- **Metal 2 3D Acceleration:** Full CoreAnimation desktop compositing, window management, and hardware acceleration via `AppleIntelTGLGraphicsMTLDriver`.
- **Dynamic VRAM:** Accurately allocated and managed up to 1536 MB.
- **Internal Panel & Backlight:** Native 1080p 60Hz timing with brightness control via keyboard shortcuts and Control Center.
- **Display & System Sleep:** Reliable display blanking, sleep, and wake cycles without ring hangs.
- **HEVC Hardware Decoding:** Full 8-bit and 10-bit HEVC decode up to 4K in QuickTime and system media players.
- **Web Video Playback:** Chromium-based browsers and Firefox support video playback up to 4K.
- **Automatic GPU Hang Recovery:** Recovers within seconds from unsupported Metal shader or video decode faults via `-wex-reset-restore`.

### Limitations
- **Safari 4K VP9/AV1:** High-resolution VP9/AV1 streams in Safari fall back to CPU decode or drop frames. Use Chrome or Firefox for 4K60.
- **H.264 Media Decode:** Dropped via `-wexgvadrop` to route H.264 through CPU software decode, avoiding donor VA driver VCS0 ring faults.

---

## OpenCore Configuration

### 1. Kext Load Order

Place the following kexts in `EFI/OC/Kexts` and register them in `config.plist` under `Kernel -> Add` in this exact order:

1. `Lilu.kext` (>= 1.7.2)
2. [`WhateverXe.kext`](https://github.com/NebbaGiveUp/WhateverXe-AlderLake/releases)
3. `AppleIntelTGLCompat.kext`
4. `AMFIPass.kext` (v1.4.1 mod required for Tahoe userspace driver validation)

### 2. Kernel Quirks

Under `Kernel -> Quirks`:
- `DisableLinkeditJettison`: `true`

### 3. NVRAM Settings

Under `NVRAM -> Add -> 7C436110-AB2A-4BBB-A880-FE41995C9F82`:

- **`boot-args`:**
  ```text
  keepsyms=1 -allow3d -disablegfxfirmware -igfxdvmt -wexgvadrop=IOGVAScaler,IOGVABGRAEnc,IOGVAHEVCEncodeCapabilities,IOGVACodec -wex-reset-restore -wex-no-multicontext
  ```

| Argument | Purpose |
| :--- | :--- |
| `-allow3d` | Enables 3D acceleration on Alder Lake GPUs. |
| `-disablegfxfirmware` | Disables loading of Intel GuC/HuC firmware. |
| `-igfxdvmt` | Bypasses DVMT pre-allocated memory check. |
| `-wexgvadrop=...` | Drops unstable Tiger Lake VA driver keys on Alder Lake, preventing VCS0 ring hangs while preserving HEVC hardware acceleration. |
| `-wex-reset-restore` | Restores RCS registers and sweeps event timeouts upon GPU hardware reset (`GDRST`). |
| `-wex-no-multicontext` | Routes compute tasks through primary render engine to prevent multi-context deadlocks. |

- **`csr-active-config`:**
  ```text
  03080000
  ```
  *(SIP disabled / `0x803`, required for userspace graphics bundles).*

### 4. DeviceProperties (`PciRoot(0x0)/Pci(0x2,0x0)`)

If BIOS DVMT Pre-Allocated is under 96 MB (Dell defaults to 32 MB or 64 MB), inject stolen memory:

```xml
<key>PciRoot(0x0)/Pci(0x2,0x0)</key>
<dict>
    <key>AAPL,slot-name</key>
    <string>built-in</string>
    <key>built-in</key>
    <data>AA==</data>
    <key>framebuffer-patch-enable</key>
    <data>AQAAAA==</data>
    <key>framebuffer-stolenmem</key>
    <data>AAzAaw==</data>
    <key>hda-gfx</key>
    <string>onboard-1</string>
</dict>
```

> **Note:** Do not inject `AAPL,ig-platform-id` or connector overrides (`framebuffer-con*`). WhateverXe manages Gen12 connector tables dynamically.

**Display Color Profile:** In macOS *System Settings -> Displays*, set Color Profile to **Generic RGB Profile** for accurate color reproduction.

### 5. Userspace Drivers

Install **`WhateverXe-Graphics-0.4.4.pkg`** from [WhateverXe-Graphics Releases](https://github.com/b00t0x/WhateverXe-Graphics/releases/tag/v0.4.4) to deploy Metal, OpenGL, and Video Acceleration bundles into `/Library/Extensions` and `/Library/GPUBundles`.

---

## Building from Source

```bash
git clone https://github.com/NebbaGiveUp/WhateverXe-AlderLake.git
cd WhateverXe-AlderLake

# Fetch MacKernelSDK
git clone https://github.com/acidanthera/MacKernelSDK.git MacKernelSDK
git -C MacKernelSDK checkout 05094e5e88cec7caedbfb35e8449ed0db94bf95b

# Fetch Lilu headers
git clone --branch 1.7.2 --depth 1 https://github.com/acidanthera/Lilu.git build/dependencies/Lilu
mkdir -p Lilu.kext/Contents/Resources
ln -s "$PWD/build/dependencies/Lilu/Lilu/Headers" Lilu.kext/Contents/Resources/Headers
ln -s "$PWD/build/dependencies/Lilu/Lilu/Library" Lilu.kext/Contents/Resources/Library

# Test and compile
python3 tools/test.py
python3 tools/build.py --release --output build/RELEASE
```

Compiled kext output: `build/RELEASE/WhateverXe.kext`.

---

## Credits

- **[b00t0x](https://github.com/b00t0x):** WhateverXe and WhateverXe-Graphics userspace bundle framework.
- **Thatisjigen:** Initial eDP mode 0 hand-off, single-output CDCLK routing, and aperture alignment patches.
- **renedis:** Display interrupt and DE master sleep/wake management.
- **[retropineaple](https://github.com/retropineaple):** GPU hardware reset interception and RCS state recovery.
- **[Acidanthera](https://github.com/acidanthera):** Lilu and MacKernelSDK.

---

## License

Licensed under the Thou Shalt Not Profit License 1.0 / Apple Public Source License. See `LICENSE.txt`.
