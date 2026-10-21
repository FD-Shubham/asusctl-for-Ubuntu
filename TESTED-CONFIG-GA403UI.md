# Tested Configuration: ASUS ROG Zephyrus G14 2024 (GA403UI)

A start-to-finish set of real-world fixes confirmed working on this specific machine, beyond the general `asusctl` build/install steps in the main [README](README.md). If you have this exact model (or a similar Ryzen + NVIDIA Optimus/MUX ROG laptop with an OLED panel), this is the order to work through things in.

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

**Before you start — GPU/Optimus base setup:** the NVIDIA Optimus/hybrid-graphics setup on this machine (kernel boot params like `nvidia_drm.modeset=1`, `nvidia.NVreg_DynamicPowerManagement=0x02`) was originally done by following a third-party GPU setup guide, not reproduced here — this doc picks up from there. If you're setting up from a bare Ubuntu install, get NVIDIA Optimus working first (`nvidia-smi` shows the dGPU, `nvidia-drm.modeset=1` in `/proc/cmdline`) before working through the items below.

---

## 1. Boot screen shows raw error text instead of a splash

**Symptom:** boot "stops" and shows plain error text instead of a clean splash screen.

**Root cause:** `/etc/default/grub` had `GRUB_CMDLINE_LINUX_DEFAULT="quiet ..."` with no `splash`. Without `splash`, Plymouth never takes over the console, so real (but harmless) kernel boot messages print as plain text — mostly firmware/DSDT bugs common across ASUS ROG boards (references to a WWAN slot / network chip this SKU doesn't physically have, a duplicate WLAN power-state declaration, a missing thermal method) and a known USB-C UCSI controller init-timeout warning that hits many 2023-2025 laptops. All cosmetic — not fixable from the OS side.

**Fix:**
```bash
sudo nano /etc/default/grub
```
Add `splash` back to `GRUB_CMDLINE_LINUX_DEFAULT`. Also worth setting `GRUB_TIMEOUT_STYLE=menu` / `GRUB_TIMEOUT=3` instead of `hidden`/`0`, so the GRUB menu (and fallback kernels) stays reachable if a future boot ever goes wrong.
```bash
sudo update-grub
```

**Verify:** `cat /proc/cmdline` shows `splash`.

---

## 2. Kernel & NVIDIA driver: moving off a frozen manual install

**Starting point this was recovering from:** a manually installed, unofficial mainline kernel build (installed because this 2024 hardware was too new for Ubuntu 24.04's launch kernel at the time) paired with a `.run`-installed NVIDIA driver — neither apt-managed, `canonical-livepatch` reporting the kernel `unsupported`. If your setup already uses the standard HWE kernel and an apt-installed `nvidia-driver-5xx`, skip to section 3.

**Target:** `linux-generic-hwe-24.04` (current, security-patched, livepatch-eligible) + `nvidia-driver-580` (apt-managed, has the Kbuild fix described above).

**The blocker:** simply installing the new kernel packages alongside the old one is safe (nothing is removed), but the post-install dkms hook will try to rebuild the *old* NVIDIA driver against the *new* kernel's headers — and if that old driver predates the `ccflags-y` migration, the build fails, leaving some kernel packages stuck "unconfigured" in dpkg (not corrupted, just mid-postinst). This is harmless as long as you haven't rebooted into the new kernel yet.

**Fix, in order:**
1. Install the new kernel packages: `sudo apt install linux-generic-hwe-24.04` (do **not** reboot yet).
2. Cleanly remove the old manually-installed NVIDIA driver with its own uninstaller (tracks exactly what the `.run` installer placed) — keep the original `.run` file around as a rollback path in case anything goes wrong:
   ```bash
   sudo /usr/bin/nvidia-uninstall -s
   sudo dkms remove nvidia/<old-version> --all   # safety net for any lingering registration
   ```
3. Install the apt-packaged driver:
   ```bash
   sudo apt install -y nvidia-driver-580
   ```
4. Resolve the packages left "unconfigured" from step 1 — this re-triggers the dkms autoinstall hook, which should now succeed against both kernels since the new driver's Kbuild is fixed:
   ```bash
   sudo dpkg --configure -a
   ```
5. Before rebooting, confirm the new driver built against **both** kernels:
   ```bash
   dkms status   # should show nvidia/580.173.02 installed for both the old and new kernel
   ```
6. Reboot, then verify:
   ```bash
   uname -r        # new HWE kernel
   nvidia-smi       # driver 580.173.02, dGPU detected
   dkms status      # installed for the running kernel
   ```

**Rollback path if anything goes wrong before rebooting:** the current (old) kernel stays installed and selectable from the GRUB menu regardless of driver outcome, and re-running the original `.run` installer with `sudo bash` restores the exact previous manual-install state.

**A new issue this can introduce:** `nvidia-driver-580` enables `nvidia-persistenced.service`, which a `.run`-based install typically never sets up. It can get tangled in a systemd ordering cycle with the save/restore unit for the phantom `nvidia_0` backlight device (see Brightness, below) via `basic.target`/`sockets.target`. Non-fatal — boot completes fine — but logs a cycle warning every boot. Since `nvidia_0` is already a known-dead backlight device on this hardware, just stop systemd from tracking it:
```bash
sudo systemctl mask 'systemd-backlight@backlight:nvidia_0.service'
```
This leaves the real `systemd-backlight@backlight:amdgpu_bl*.service` untouched.

**Once stable on the new kernel:** remove the old kernel packages if you no longer want the fallback, and re-enable livepatch if it was disabled during the migration:
```bash
sudo canonical-livepatch enable
```

**Also worth checking (inert under Wayland, revisit only if you ever fall back to Xorg):** a `.run`-based NVIDIA install can leave `/etc/X11/xorg.conf.d/10-nvidia-brightness.conf` set to `Option "Backlight" "nvidia_0"` (the same known-dead device), and a static BusID-pinned `/etc/X11/xorg.conf` generated by `nvidia-xconfig` — generally not what you want on a hybrid Optimus laptop. Confirm you're on Wayland first: `echo $XDG_SESSION_TYPE`.

---

## 3. `asusd-user` crashes on every login

**Symptom:** `asusd-user.service` (the GNOME user-session component) fails with a core-dump every login.

**Root cause has two layers:**
1. A leftover unit override — some setup guides drop `/etc/xdg/systemd/user/asusd-user.service`, pointing at a stale hand-built binary at `/usr/local/bin/asusd-user` instead of the real apt-installed `/usr/bin/asusd-user`.
2. **The correct package binary crashes too**, on laptops with more than one Aura-lighting-capable device (e.g. keyboard + a second lighting zone). Confirmed by running `/usr/bin/asusd-user --version` directly: it panics with a D-Bus `MethodError` on `Unknown object '/xyz/ljones/Aura'`. The running `asusd` daemon exposes **per-device** Aura D-Bus paths (e.g. `/xyz/ljones/aura/<id1>` and `/xyz/ljones/aura/<id2>`), but this version of `asusd-user` hardcodes the old singular path. This is an **upstream asusctl bug**, not fixable via config — check the [asusctl releases page](https://gitlab.com/asus-linux/asusctl/-/releases) periodically for a fix if your laptop has multiple Aura zones.

**Fix (mitigation, not a real fix):**
```bash
sudo rm -f /etc/xdg/systemd/user/asusd-user.service
sudo rm -f /usr/local/bin/asusd-user
systemctl --global mask asusd-user.service
```
This stops the crash-loop/log spam. Fan curves, power profiles, and keyboard backlight brightness (via `asusctl -k` / the system `asusd.service`) all keep working — only the user-level tray/notification helper (Aura sync notifications, etc.) is disabled.

**To retry in future:** if a newer `.deb`/apt version fixes multi-device Aura enumeration, install it, then `sudo systemctl --global unmask asusd-user.service`, and test `asusd-user --version` no longer panics before re-enabling.

---

## 4. `rog-control-center` (tray GUI) doesn't start automatically, or behaves oddly

**Two separate issues, same root pattern as the `asusd-user` crash above:**

1. **No autostart entry at all** — the GUI never launches at login unless started manually.
2. **Stale-binary shadowing**: a correct package binary at `/usr/bin/rog-control-center` can be shadowed in `$PATH` by an older hand-built copy at `/usr/local/bin/rog-control-center` left over from an earlier manual build. The stale one runs without hard-crashing but logs `Multiple asusd interfaces devices found` and repeated `Attribute with no name, skipping` — a degraded/version-mismatched state, same symptom family as the `asusd-user` bug. The correct `/usr/bin` one only logs two harmless `WARN` lines about supergfxd version and no dGPU status.

**Fix:**
```bash
sudo rm -f /usr/local/bin/rog-control-center
```
Then create `~/.config/autostart/rog-control-center.desktop` as a copy of the package's `/usr/share/applications/rog-control-center.desktop`, with `X-GNOME-Autostart-enabled=true` added. Since its `Exec=rog-control-center` line is a bare `$PATH` lookup (no hardcoded path), removing the stale binary makes it resolve to the correct one automatically.

Check `~/.config/rog/rog-control-center.cfg` has `startup_in_background: true` and `enable_tray_icon: true` if you want it to start silently in the tray with no popup window. Takes effect at next login.

3. **Shadow binary far enough ahead of the daemon causes a hard crash-loop, not just degraded logging.** If the shadowing binary in `/usr/local/bin` is *newer* than the installed `asusd` (e.g. a hand-built `rog-control-center` 6.1.15 against an apt-installed `asusd` 6.1.12), the D-Bus schema drift goes past "degraded" into an outright panic: `Io error: Broken pipe (os error 32)` followed by `thread 'main' panicked ... Error running winit event loop: Exit Failure: 1`. GNOME's session manager relaunches the autostart app scope every time it dies — so this shows up as unexplained, recurring crash/tray notifications throughout the day rather than a one-time glitch. `journalctl --since "-1d" | grep rog-control-center.desktop` will show the repeating `Version mismatch: asusctl = X, asusd = Y` line right before each crash.

**Fix:** get `asusd` / `asusctl` / `asusd-user` / `rog-control-center` back onto one matching version — ideally current upstream rather than whatever's in Ubuntu's apt repo (apt lags, and a stale local `git` checkout can lag too):
```bash
# heads up: this project migrated off GitLab in mid-2026 — gitlab.com/asus-linux/asusctl
# is now a frozen mirror stuck at 6.3.8. Current development + releases live at
# https://github.com/OpenGamingCollective/asusctl
cd asusctl   # your existing checkout — add a new remote rather than losing origin's history
git remote add github https://github.com/OpenGamingCollective/asusctl.git
git fetch --tags github
git checkout <latest-tag>   # e.g. 6.5.0 — check the repo's Releases page for current
make build
sudo make install
sudo rm -f /usr/local/bin/rog-control-center   # remove any older shadow copy, same as above
sudo systemctl daemon-reload
sudo systemctl restart asusd
```

**Verify:** `journalctl -u asusd --no-pager | grep "daemon v"` shows the same new version for every component (`rog-anime`, `rog-slash`, `rog-aura`, `rog-profiles`, `rog-platform`), and relaunching `rog-control-center` logs no `Version mismatch`, `Multiple asusd interfaces devices found`, or `Io error: Broken pipe` lines.

---

## 5. Brightness keys stop working intermittently

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

**Known residual gap:** this only covers the physical F7/F8 keys via the custom-keybinding mechanism. The GNOME on-screen brightness slider (Quick Settings / Settings > Displays) goes through `logind`'s own `SetBrightness` D-Bus call, which does its own device selection among `amdgpu_bl*` vs the phantom `nvidia_0` backlight device — not covered by this fix. Not observed as an actual problem in practice, but worth knowing about.

**Verify:** `~/.local/bin/panel-brightness info` should report whichever `amdgpu_bl*` device exists that boot; F7/F8 should visibly dim/brighten the screen regardless of the device number.

---

## 6. MUX / GPU mode switching (`supergfxctl`)

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

**Fan curves (`asusctl fan-curve`):** three power profiles (`Quiet`/`Balanced`/`Performance`), each with a firmware-default CPU/GPU/MID curve. By default, custom curve control is disabled on all three — the EC's own firmware fan logic drives the fans, not `asusd`.
```bash
asusctl profile -P <Quiet|Balanced|Performance>       # switch profile (or -n to cycle)
asusctl fan-curve -m <profile> -e true                # enable custom curve control
asusctl fan-curve -m <profile> -f <cpu|gpu|mid> -D <temp:percent,...>   # e.g. 30c:20%,50c:30%,65c:45%,75c:65%,85c:90%,95c:100%
asusctl fan-curve -m <profile> -d                     # revert to firmware default
```

---

## 7. Suspend: laptop stays warm / wakes immediately after closing the lid

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

## 8. General system reliability (not G14-specific, but bit us alongside the above)

**Automatic security updates can look "on" while silently doing nothing.** `unattended-upgrades.service` showing `enabled` in systemd is not proof it's doing anything — check its actual trigger config:
```bash
cat /etc/apt/apt.conf.d/20auto-upgrades
```
If `Update-Package-Lists` or `Download-Upgradeable-Packages` is `"0"`, the service has nothing to act on despite looking enabled. Cross-check with `/var/log/apt/history.log*` for when unattended-upgrade activity actually last ran. On this machine, some earlier step in the GPU/kernel setup process had silently disabled this — plausibly to stop apt from pulling a newer kernel and breaking a fragile manual `.run`-driver setup (the exact failure mode in section 2 above). Once you're on an apt-managed driver + a real HWE kernel, it's safe to restore:
```bash
sudo tee /etc/apt/apt.conf.d/20auto-upgrades <<'EOF'
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Unattended-Upgrade "1";
EOF
```

**Also check for orphaned kernel debris** from earlier abandoned kernel-migration attempts — headers/modules for kernel versions with no matching bootable image (`apt-mark showmanual` + cross-check against `dpkg -l 'linux-image-*'`), and orphaned `/boot/initrd.img-*` files with no matching installed kernel package. These are pure dead weight and safe to purge with `sudo apt autoremove` once confirmed orphaned.

**Nice-to-have, not a fix:** a battery charge limit for longevity, e.g. `asusctl -c 80`.

---

## 9. GNOME `Battery-Health-Charging` extension fights `asusd` and spams polkit prompts

**Symptom:** unexplained authentication activity (polkit prompts, or a fingerprint-daemon cycle if your prompt flow auto-tries fingerprint) roughly every 20-40 minutes, all day, for no apparent reason.

**Root cause, two layers:**
1. **Redundant with `asusd`.** The popular [Battery-Health-Charging](https://github.com/maniacx/Battery-Health-Charging) GNOME extension has Asus-specific support that writes directly to `/sys/class/power_supply/BAT*/charge_control_end_threshold` — the exact same sysfs knob `asusd` already manages natively (`asusctl -c <20-100>`, or the slider in `rog-control-center`). Two independent, unaware-of-each-other writers on one knob is the same conflict class as `power-profiles-daemon` vs `asusd` above — whichever ran most recently wins, silently.
2. **Its own polkit rule can go missing.** The extension ships a rule (`10-dem.batteryhealthcharging.setthreshold.rules`) meant to let its periodic `CHECKINSTALLATION` self-check run via `pkexec` without prompting. If that rule isn't actually present under `/etc/polkit-1/rules.d` or `/usr/share/polkit-1/rules.d` — confirm with `find /etc/polkit-1 /usr/share/polkit-1 -iname '*batteryhealth*'` — every periodic check falls back to the generic `org.freedesktop.policykit.exec` action and forces a real authentication. Visible in `journalctl` as a `pkexec ... CHECKINSTALLATION ...` line plus an `fprintd`/polkit auth cycle, repeating on the extension's own internal timer all day.

**Fix:** since `asusd`/`rog-control-center` already own charge-limit control on ASUS hardware, just disable (or fully remove) the extension rather than fixing its polkit rule:
```bash
gnome-extensions disable Battery-Health-Charging@maniacx.github.com
# fully remove instead, if you don't need it for some other non-Asus device:
# rm -rf ~/.local/share/gnome-shell/extensions/Battery-Health-Charging@maniacx.github.com
# sudo rm -f /usr/local/bin/batteryhealthchargingctl-$USER
```

**Verify:** `journalctl --since "-1h" | grep CHECKINSTALLATION` stays empty going forward, and the charge limit is still enforced — `cat /sys/class/power_supply/BAT*/charge_control_end_threshold` matches whatever you set with `asusctl -c`.

---

## 10. Touchpad settings (GNOME)

These are the GNOME touchpad settings in use on this unit. They live in dconf, not in any config file, so a fresh install or new user account loses them. Nothing here is G14-specific; it's just what was tuned.

Notable choices: `accel-profile 'flat'` (no pointer acceleration), `speed 0.2`, natural scrolling, tap-to-click and tap-and-drag on, edge scrolling off, `click-method 'areas'` (clicks depend on where you press, not finger count).

```bash
gsettings set org.gnome.desktop.peripherals.touchpad accel-profile 'flat'
gsettings set org.gnome.desktop.peripherals.touchpad click-method 'areas'
gsettings set org.gnome.desktop.peripherals.touchpad disable-while-typing true
gsettings set org.gnome.desktop.peripherals.touchpad edge-scrolling-enabled false
gsettings set org.gnome.desktop.peripherals.touchpad left-handed 'mouse'
gsettings set org.gnome.desktop.peripherals.touchpad middle-click-emulation false
gsettings set org.gnome.desktop.peripherals.touchpad natural-scroll true
gsettings set org.gnome.desktop.peripherals.touchpad send-events 'enabled'
gsettings set org.gnome.desktop.peripherals.touchpad speed 0.2
gsettings set org.gnome.desktop.peripherals.touchpad tap-and-drag true
gsettings set org.gnome.desktop.peripherals.touchpad tap-and-drag-lock false
gsettings set org.gnome.desktop.peripherals.touchpad tap-button-map 'default'
gsettings set org.gnome.desktop.peripherals.touchpad tap-to-click true
gsettings set org.gnome.desktop.peripherals.touchpad two-finger-scrolling-enabled true
```

**Verify:** `gsettings list-recursively org.gnome.desktop.peripherals.touchpad` matches the values above. Changes apply immediately, with no logout needed. To back up or restore everything at once: `dconf dump /org/gnome/desktop/peripherals/touchpad/ > touchpad.dconf` and `dconf load /org/gnome/desktop/peripherals/touchpad/ < touchpad.dconf`.

---

## 11. F6 as a screenshot key (GNOME)

On the G14 the F6 key is labelled as the screen-snip key, but nothing is bound to it out of the box. GNOME's screenshot UI (region / window / full screen picker) is bound to `Print` only. Add F6 alongside it:

```bash
gsettings set org.gnome.shell.keybindings show-screenshot-ui "['Print', 'F6']"
```

This lives in dconf, so like the touchpad settings it is lost on a fresh install or new user account. Changes apply immediately, with no logout needed.

**Verify:** `gsettings get org.gnome.shell.keybindings show-screenshot-ui` prints `['Print', 'F6']`, and pressing F6 opens the screenshot UI. If nothing happens, try Fn+F6: depending on the Fn-lock state the F-row sends media keys by default. Also check F6 isn't already taken by a custom shortcut (`gsettings get org.gnome.settings-daemon.plugins.media-keys custom-keybindings`, then read each `binding`).

To undo: `gsettings reset org.gnome.shell.keybindings show-screenshot-ui`.

---

## Changelog

- Folded in the full setup/fix history for this GA403UI unit: boot splash (§1), kernel+driver migration off a frozen manual install (§2), `asusd-user` crash (§3), `rog-control-center` autostart (§4), brightness (§5), MUX/fan curves (§6), suspend (§7), and general update/reliability hygiene (§8).
- Added the hard-crash-loop variant of the `rog-control-center` shadow-binary bug (§4.3), noting the project's mid-2026 migration from GitLab to `github.com/OpenGamingCollective/asusctl` and a full version-matched rebuild fix.
- Added §9: the `Battery-Health-Charging` GNOME extension's Asus support duplicating `asusd`'s charge-limit control and, when its polkit rule isn't installed, causing recurring unexplained authentication prompts every 20-40 minutes.
- Added §10: GNOME touchpad settings (flat accel, speed 0.2, natural scroll, tap-to-click, `areas` click method) with reproducible `gsettings` commands.
- Added §11: binding F6 to GNOME's screenshot UI via `gsettings` (alongside `Print`).
