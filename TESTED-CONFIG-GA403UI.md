# Tested Configuration: ASUS ROG Zephyrus G14 2024 (GA403UI)

Real-world fixes confirmed working on this specific machine, beyond the general install steps in the main [README](README.md). If you have this exact model (or a similar Ryzen + NVIDIA Optimus/MUX ROG laptop with an OLED panel), start here.

## Hardware / software this was validated on

| Component | Version |
|---|---|
| Laptop | ASUS ROG Zephyrus G14 2024 (GA403UI) |
| CPU / iGPU | Ryzen 9 8945HS / Radeon 780M (AMD Phoenix3) |
| dGPU | RTX 4070 Max-Q (NVIDIA, Optimus/Hybrid, **real hardware MUX**) |
| Panel | OLED (does **not** implement working ACPI `_BCM`/`_BQC` backlight methods — see Brightness section) |
| OS | Ubuntu 24.04.4 LTS |
| Kernel | `linux-generic-hwe-24.04`, validated across `7.0.0-28-generic` → `7.0.0-30-generic` |
| NVIDIA driver | `nvidia-driver-580` (580.173.02) from Ubuntu's `noble-updates/restricted` repo — apt-managed, not the `.run` installer |
| asusctl / asusd | 6.1.12-1 |
| supergfxctl | 5.2.7 |

**Why this driver/kernel combo specifically:** NVIDIA driver branches before 580.x use the legacy `EXTRA_CFLAGS` Kbuild variable for their include path. Linux 6.15+ dropped compatibility shimming for `EXTRA_CFLAGS`, so on HWE kernels built past that cutoff the driver's dkms build silently drops its include path and fails with `fatal error: os-interface.h: No such file or directory` — a missing `-I` flag, not a missing file. `580.173.02`'s Kbuild uses `ccflags-y` instead, which still works. Confirmed by reading the actual Kbuild source, not inferred from the version number alone — some early 580.x point releases have been reported to still fail against newer HWE kernels, so this exact build is what's known-good here.

---

## 1. Brightness keys stop working intermittently

**Symptom:** F7/F8 brightness keys work on some boots, silently do nothing on others.

**Do not use `acpi_backlight=vendor`/`acpi_backlight=video`/`amdgpu.backlight=0` for this fix on an OLED panel like this one** — tested directly (set 10% then 90%, no visible change), this panel's ACPI backlight interface accepts writes but has zero effect on the hardware.

**Real root cause:** the `amdgpu_bl*` backlight device's numeric suffix (`amdgpu_bl0`, `_bl1`, `_bl2`, ...) is **not stable across boots** — it depends on a driver-probe-order race between `amdgpu` and `nvidia` at boot. Any brightness shortcut that hardcodes a specific device name (e.g. `brightnessctl -d amdgpu_bl2 set +5%`) will silently fail — device not found, no visible error — on any boot where the device enumerates under a different number.

**Fix:** resolve the device name dynamically instead of hardcoding it.

`~/.local/bin/panel-brightness`:
```bash
#!/usr/bin/env bash
set -euo pipefail
dev=$(ls /sys/class/backlight/ | grep -m1 '^amdgpu_bl' || true)
if [ -z "$dev" ]; then
    notify-send "Brightness control" "No amdgpu backlight device found" 2>/dev/null || true
    exit 1
fi
exec brightnessctl -d "$dev" "$@"
```

Point your F7/F8 GNOME custom keyboard shortcuts at it instead of a raw `brightnessctl -d <device>` call:
```
/home/<you>/.local/bin/panel-brightness set +5%
/home/<you>/.local/bin/panel-brightness set 5%-
```

**Known residual gap:** this only covers the physical F7/F8 keys via the custom-keybinding mechanism. The GNOME on-screen brightness slider (Quick Settings / Settings > Displays) goes through `logind`'s own `SetBrightness` D-Bus call, which does its own device selection among `amdgpu_bl*` vs a phantom `nvidia_0` backlight device — not covered by this fix. Not observed as an actual problem in practice, but worth knowing about.

**Verify:** `~/.local/bin/panel-brightness info` should report whichever `amdgpu_bl*` device exists that boot; F7/F8 should visibly dim/brighten the screen regardless of the device number.

---

## 2. MUX / GPU mode switching (`supergfxctl`)

This laptop has a **real hardware MUX**, not just software Optimus. Supported modes (confirmed via `supergfxctl -s`):

| Mode | Behavior |
|---|---|
| `Integrated` | dGPU off, AMD iGPU only |
| `Hybrid` | Optimus/PRIME render offload (default) |
| `AsusMuxDgpu` | Panel physically routed to the RTX 4070, bypassing the AMD iGPU entirely |

Switch with:
```bash
sudo supergfxctl -m <mode>
```

If `/etc/supergfxd.conf` has `always_reboot: false`, a logout (within the ~180s window) is enough — no full reboot needed.

**Caveat:** the brightness fix above (`panel-brightness`) only looks for `amdgpu_bl*` devices. That's correct in `Hybrid` and moot in `Integrated`, but **would silently fail in `AsusMuxDgpu` mode**, since the AMD iGPU isn't driving the panel there and `amdgpu_bl*` may not exist. Only worth fixing if you actually use `AsusMuxDgpu` day-to-day — if so, generalize the wrapper to detect whichever backlight device is real for the currently active `supergfxctl` mode.

---

## 3. Suspend: laptop stays warm / wakes immediately after closing the lid

**Symptom:** laptop suspended via lid-close or idle wakes back up almost immediately (observed: a suspend cycle that lasted only ~2 seconds), then stays fully awake and warm for hours in a bag.

**Root causes identified:**
1. **s2idle only, no S3 deep sleep** (`/sys/power/mem_sleep` shows only `[s2idle]`) — a BIOS/firmware limitation on this ASUS model, not fixable from the OS. s2idle keeps the CPU/GPU/PCIe partially active, which is inherently warmer than S3.
2. **Spurious PCIe PME wakeups** from the AMD PCIe bridge (`pcieport 0000:00:08.1`/`.3: PME: Spurious native interrupt!`) pulling the system straight back out of s2idle almost immediately after suspend.
3. **ACPI lid switch non-compliance** (`ACPI: button: The lid device is not compliant to SW_LID`) — can cause the lid-close event to not reliably trigger suspend.
4. **Browser media inhibitors** (e.g. Firefox holding a "playing video" suspend inhibitor from a background tab) can also silently block suspend from triggering at all.

**Fix applied and confirmed working:** disable the specific spurious PCIe wake sources via a oneshot systemd service, plus kernel parameters to make s2idle behave better on this AMD platform.

`/etc/systemd/system/disable-pcie-wakeup.service`:
```ini
[Unit]
Description=Disable spurious PCIe wakeup sources for reliable s2idle
After=multi-user.target

[Service]
Type=oneshot
ExecStart=/bin/bash -c 'echo GPP0 > /proc/acpi/wakeup || true'
ExecStart=/bin/bash -c 'echo GPP5 > /proc/acpi/wakeup || true'
ExecStart=/bin/bash -c 'echo GPP6 > /proc/acpi/wakeup || true'
ExecStart=/bin/bash -c 'echo GP11 > /proc/acpi/wakeup || true'
ExecStart=/bin/bash -c 'echo NHI0 > /proc/acpi/wakeup || true'
ExecStart=/bin/bash -c 'echo XHC0 > /proc/acpi/wakeup || true'
ExecStart=/bin/bash -c 'echo XHC1 > /proc/acpi/wakeup || true'
ExecStart=/bin/bash -c 'echo XHC3 > /proc/acpi/wakeup || true'
ExecStart=/bin/bash -c 'echo XHC4 > /proc/acpi/wakeup || true'
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now disable-pcie-wakeup.service
```

> `/proc/acpi/wakeup` is a toggle file — writing a device name flips its enabled/disabled state. Only disable devices that show as `*enabled` on your own system first (`cat /proc/acpi/wakeup`); the exact device list can differ by unit/board revision.

In `/etc/default/grub`, add to `GRUB_CMDLINE_LINUX_DEFAULT`:
```
acpi.ec_no_wakeup=1 amd_pmc.disable_workarounds=1
```
then `sudo update-grub`.

- `acpi.ec_no_wakeup=1` — stops the Embedded Controller from generating spurious wakeups during s2idle.
- `amd_pmc.disable_workarounds=1` — disables AMD PMC workarounds known to interfere with clean s2idle on Zen4 platforms.

**Also worth doing:** enable idle-suspend on AC power as a fallback, in case the lid switch's ACPI non-compliance ever causes a missed lid-close event:
```bash
gsettings set org.gnome.settings-daemon.plugins.power sleep-inactive-ac-type 'suspend'
gsettings set org.gnome.settings-daemon.plugins.power sleep-inactive-ac-timeout 1200
```

**Verify:**
```bash
cat /proc/acpi/wakeup                              # confirm the listed devices are now disabled
systemctl suspend                                  # test manually
journalctl -b -u systemd-suspend.service --no-pager
journalctl -b -k | grep -i spurious                # should be empty/gone
```

---

## Changelog

- Documented brightness, MUX, and suspend fixes tested on this GA403UI unit; noted tested `nvidia-driver-580` (580.173.02) and HWE kernel range `7.0.0-28` → `7.0.0-30`.
