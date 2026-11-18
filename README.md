# WhateverXe (Alder Lake-P / Dell eDP Patch)

This repository is a rebuild of WhateverXe with critical display pipeline and firmware hand-off patches for 12th-Generation Intel Iris Xe Graphics (`8086:46A6`, Alder Lake-P) on macOS Tahoe and macOS Sequoia.

It has been tested on a **Dell Latitude 7430** with an **Intel Core i7-1270P** processor running macOS Tahoe 26.7.3 (`25G342`), giving full Metal 2 acceleration, native backlight control, and hardware-accelerated WindowServer rendering.

---

## Why Official WhateverXe Failed on Alder Lake-P Laptops

When running official WhateverXe releases (v0.8.0 through v0.8.2) on 12th-Gen Dell laptops such as the Latitude 7430 or Precision 5570, the system boots, but display acceleration fails completely:

- macOS System Information reports: `IrisXe Graphics VRAM (Total): 7 MB`, `Kernel Extension Info: No Kext Loaded`, and no Metal support.
- The display remains stuck on unaccelerated software rendering with severe window lagging and tearing.
- In IORegistry, `display0` remains attached to the firmware VESA fallback framebuffer `.Display_boot` (`IONDRVFramebuffer`).
- `IntelAccelerator` fails to attach under `IGPU@2`.

### Root Causes

1. **eDP Boot Port Mode 0 Hand-off:**  
   Dell laptop firmware hands off the internal embedded DisplayPort (eDP) in native port mode 0. Official WhateverXe checks the boot port against desktop and Tiger Lake expectations and drops or rejects the port in mode 0. Consequently, the native framebuffer driver (`AppleIntelFramebuffer@0`) cannot claim the internal panel.

2. **WindowServer Trapped on `.Display_boot`:**  
   In the macOS graphics subsystem, `IntelAccelerator` (`AppleIntelTGLGraphics`) will only bind and initialize hardware acceleration once WindowServer successfully attaches to an active Apple framebuffer pipe. Because the framebuffer was rejected, WindowServer stays on `.Display_boot`, and the 3D acceleration stack is never brought up.

3. **Single-Output CDCLK Timing Path:**  
   When only a single display pipe (the internal eDP screen) is present without external monitors, the stock driver fails to compute the core display clock (CDCLK) properly, resulting in a black screen or dropped initialization.

4. **Aperture Alignment Check:**  
   Stock WhateverXe rejects the framebuffer aperture mapping if the BIOS-reserved stolen memory does not end on a strict 4KB page boundary.

---

## How This Implementation Works

This repository starts from the official **WhateverXe 0.8.2** release tag and integrates 10 community commits that resolve the Alder Lake-P display pipeline:

1. **eDP Port Mode 0 Admission (`dca0ff6`):**  
   Explicitly admits the eDP boot port in native port mode 0 during firmware hand-off, allowing `AppleIntelFramebuffer@0` to claim the panel.
2. **Single-Output CDCLK Routing (`490e06e`):**  
   Calculates and sets the 1305.6 MHz display clock path correctly when only the internal panel pipe has an active framebuffer.
3. **Flexible Aperture Bounds (`8321cc9`):**  
   Accepts framebuffer memory apertures that do not terminate on a page boundary.
4. **Display Sleep and Wake Interrupt Handling (`0859ce7` through `5ec4073`):**  
   - Keeps display interrupts active across single-output display wake transitions.
   - Parks the display IRQ master off while the panel pipe is powered down.
   - Maintains DE master cache coherency with underrun and interrupt storm guards.
   - Prevents the native idle timer from re-arming a parked DE master prematurely.
   - Re-arms the DE master once the display pipes resume post-wake.
5. **GVA Video Decode Diagnostic Hooks (`46530e8`, `31e3bc8`):**  
   Adds opt-in boot arguments (`-wex-no-gva-scaler` and `-wex-no-gva`) to isolate hardware video decode pipelines if needed.
6. **Selective Key-Bisection & Media Compatibility (`-wexgvadrop`):**  
   Adds `-wexgvadrop=<Key1>,<Key2>,...` boot argument support to selectively unpublish arbitrary `IOGVA*` properties from `IntelAccelerator` after driver initialization. This enables asymmetric media decoding (e.g. retaining GPU hardware HEVC decode while routing H.264 to CPU software decode) to resolve Alder Lake-P donor VA driver hangs.

---

## Hardware Profile and Test Environment

Verified on the following configuration:

| Component | Specification |
| :--- | :--- |
| Model | Dell Latitude 7430 |
| Processor | Intel Core i7-1270P (Alder Lake-P, 12 Cores / 16 Threads) |
| Integrated Graphics | Intel Iris Xe Graphics (`8086:46A6`, Revision `0x0C`, Subsystem `1028:0B09`) |
| Display Panel | 14.0-inch 1920x1080 60Hz internal eDP |
| Operating System | macOS Tahoe 26.7.3 (`25G342`) |
| SMBIOS | `MacBookPro16,2` |
| VRAM | 1536 MB Dynamic |
| Metal Support | Metal 2 (`AppleIntelTGLGraphicsMTLDriver`) |
| Display Route | `IGPU@2` -> `AppleIntelFramebuffer@0` -> `display0` -> `AppleBacklightDisplay` |

---

## What Works and Known Limitations

### What Works

- **Full Metal 2 Hardware Acceleration:** Powered by `AppleIntelTGLGraphicsMTLDriver`, providing native CoreAnimation desktop compositing, transparent menu bars, fluid dock animations, and responsive Launchpad navigation.
- **Dynamic VRAM Management:** Accurately reports and allocates 1536 MB of dynamic video memory to system processes.
- **Internal Display Timing & Backlight:** Native 1920x1080 60Hz panel timing with working backlight level adjustments via keyboard shortcuts and Control Center.
- **Display Idle Sleep and Wake:** Screen blanking due to inactivity and subsequent waking via keyboard or trackpad input operates normally without freezing or blitter ring hangs, confirmed stable on the Dell Latitude 7430.
- **System Sleep and Wake:** Full ACPI sleep (closing the lid or selecting Sleep from the Apple menu) and waking cycle reliably.
- **Hardware-Accelerated Video Playback & Media Profile:**
  - **HEVC / H.265 (8-bit and 10-bit Main/Main 10):** Full GPU hardware decode via `IOGVAHEVCDecode`, playing smoothly up to 4K resolution in QuickTime, TV app, and media players.
  - **H.264 / AVC:** Handled via CPU software decoding using the 12-core / 16-thread i7-1270P when configured with `-wexgvadrop=IOGVAScaler,IOGVABGRAEnc,IOGVAHEVCEncodeCapabilities,IOGVACodec`. This avoids the donor VA driver's hardware Multi-Format Codec (`VCS0`) ring fault (`EIR = 0x4`) and eliminates the GPU reset freeze during Telegram animated GIF/sticker playback.
  - **Fullscreen Media & Photo Viewer:** Instant opening without the 9.2-second `VPHAL` unsupported resolution timeout that previously froze messaging apps.
  - **Full Web Video Streaming:** Fully functional in Chromium-based browsers (Google Chrome, Brave, Microsoft Edge), Mozilla Firefox up to 4k
### Known Limitations

- **Safari 4K VP9/AV1 Streaming:**  
  Safari plays video streams smoothly up to 1080p60. 4K streams with VP9 or AV1 codecs rely on CPU software decoding or may fall back depending on WebKit feature flags. For 4K 60fps VP9/AV1 playback, Chromium-based browsers or Firefox are recommended.

---

## Installation and OpenCore Configuration

### 1. Required Kexts and Load Order

Dell Latitude 7430 users can look at the preconfigured EFI [Dell Latitude 7430 OpenCore EFI](https://github.com/NebbaGiveUp/Dell-Latitude-7430-Hackintosh-OC-EFI) repository, which already includes all tested fixes, kexts, and ACPI patches for this model and will help you to build yours.

For other laptops, ensure the following kexts are placed in `EFI/OC/Kexts` and added in `config.plist` under `Kernel -> Add` (Just use OC snapshot if you're using propertree gui) in this exact order:

1. `Lilu.kext` (v1.7.2 or newer)
2. [`WhateverXe.kext`](https://github.com/NebbaGiveUp/WhateverXe-AlderLake/releases/download/v0.8.2-adl/WhateverXe-0.8.2-AlderLake-RELEASE.zip) (download prebuilt binary from [Releases](https://github.com/NebbaGiveUp/WhateverXe-AlderLake/releases) or compile from source)
3. `AppleIntelTGLCompat.kext` (bundled with WhateverXe-Graphics)
4. `AMFIPass.kext` (**Must use the AMFIPass 1.4.1 Mod build**; official upstream AMFIPass will fail library validation for the userspace graphics drivers on macOS Tahoe)

### 2. Kernel Quirks

In `config.plist` under `Kernel -> Quirks`:
- `DisableLinkeditJettison`: Set to `true` (preserves runtime symbol tables required for driver linking).

### 3. NVRAM Configuration

In `config.plist` under `NVRAM -> Add -> 7C436110-AB2A-4BBB-A880-FE41995C9F82`:

- **`boot-args`:**
  ```text
  keepsyms=1 -allow3d -disablegfxfirmware -igfxdvmt -wexgvadrop=IOGVAScaler,IOGVABGRAEnc,IOGVAHEVCEncodeCapabilities,IOGVACodec
  ```
  *(The `-wexgvadrop` argument selectively unpublishes unstable Tiger Lake VA driver keys on Alder Lake-P, preventing Telegram GIF GPU hangs and fullscreen photo 9-second timeouts while retaining full GPU hardware HEVC decoding and 60Hz UI smoothness)*

- **`csr-active-config`:**
  ```text
  03080000
  ```
  *(Equivalent to `0x803`, required to allow the modified graphics bundles to run)*

### 4. DeviceProperties (`PciRoot(0x0)/Pci(0x2,0x0)`)

Check your BIOS settings for DVMT Pre-Allocated or Video Memory. If it is lower than 96 MB (Dell BIOS firmware typically pre-allocates 32 MB or 64 MB), add the following properties under `DeviceProperties -> Add -> PciRoot(0x0)/Pci(0x2,0x0)` in your `config.plist`. If your BIOS already sets DVMT to 96 MB or higher, you can skip this step.

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
### Misc.
   Set display color mode to "generic rgb" in Settings> Display for the bext experience, it fixes many rendering glitches as well if you are facing any.
   
**Important note on connector properties:**  
Do **not** inject `AAPL,ig-platform-id` or hardcoded connector overrides (`framebuffer-con0-busid`, `framebuffer-con0-enable`, `framebuffer-con0-flags`, `framebuffer-con0-type`). WhateverXe manages Gen12 connector tables dynamically via its Display13 engine. Hardcoding legacy WhateverGreen properties will break the native eDP handshake.

### 5. Userspace Driver Installation

Download and install **`WhateverXe-Graphics-0.4.4.pkg`** from the [WhateverXe-Graphics Releases](https://github.com/b00t0x/WhateverXe-Graphics/releases/tag/v0.4.4). This package installs the required Metal, OpenGL, and Video Acceleration bundles into `/Library/Extensions` and `/Library/GPUBundles`.

---

## Power Management Notes

Thanks to the WhateverXE community commits used in this build, standard display sleep timeouts work reliably out of the box on the Dell Latitude 7430. You do not need to alter power management settings for daily use.

If you are adapting this build for another laptop model and encounter blitter ring hangs during screen blanking, you can disable display idle sleep in Terminal as a troubleshooting measure:

```bash
sudo pmset -a displaysleep 0
```

---

## Building from Source

To build `WhateverXe.kext` locally:

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

# Run tests and build Release
python3 tools/test.py
python3 tools/build.py --release --output build/RELEASE
```

The compiled `WhateverXe.kext` will be placed in `build/RELEASE/WhateverXe.kext`.

---

## Prebuilt Binaries

Precompiled RELEASE binaries matching this repository can be downloaded from the [Releases](https://github.com/NebbaGiveUp/WhateverXe-AlderLake/releases) tab.

---

## Attribution and Upstream Credits

This project is not a fork from scratch, nor does it seek to replace upstream WhateverXe. It exists solely to provide a maintained, fully functioning reference implementation for Alder Lake-P laptop users facing internal eDP display hand-off failures.

All core architecture, driver shims, and Gen12 Xe-LP reverse engineering belong to the original authors:

- **b00t0x and the WhateverXe Project:** Original author and maintainer of [b00t0x/WhateverXe](https://github.com/b00t0x/WhateverXe) and [WhateverXe-Graphics](https://github.com/b00t0x/WhateverXe-Graphics). The entire concept of leveraging Apple's unreleased Tiger Lake 16.0.32 graphics driver on modern macOS is their work.
- **Thatisjigen:** Authored the fundamental eDP hand-off, single-output CDCLK routing, and framebuffer aperture alignment patches (WhateverXe Issue/PR #38).
- **renedis:** Maintained and ported the patchset to the WhateverXe 0.8.x tree, authored the display interrupt and Display Engine (DE) sleep/wake master parking logic, and validated it on Dell Precision hardware (WhateverXe Issue #90, `precision-5570-display-wake` branch).
- **Acidanthera:** Authors of Lilu and MacKernelSDK.

---

## License

This project is licensed under the same terms as upstream WhateverXe (Thou Shalt Not Profit License 1.0 / Apple Public Source License). See `LICENSE.txt` for details.
