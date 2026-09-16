# asusctl-for-Ubuntu
Porting asusctl (made for Fedora) for Ubuntu

> **Have an ASUS ROG Zephyrus G14 2024 (GA403UI) or similar Ryzen + NVIDIA MUX laptop with an OLED panel?**
> See [`TESTED-CONFIG-GA403UI.md`](TESTED-CONFIG-GA403UI.md) for a full start-to-finish setup/fix reference: boot splash, the tested kernel/NVIDIA-driver combo (with migration steps off a frozen manual install), `asusd-user` and `rog-control-center` crash workarounds, intermittent brightness keys, MUX/GPU mode switching + fan curves, suspend not staying asleep, and general update hygiene.

Installing asusctl on Ubuntu

asusctl is a fantastic utility for ASUS ROG/TUF laptops, but it is primarily packaged for Fedora and Arch. Upstream's own README now includes a bare-bones "Ubuntu and Pop!_OS (Unsupported)" build recipe too, but it stops at `make install`. This guide covers everything upstream doesn't: the full dependency list needed to build `rog-control-center` (the GUI) as well as `asusd`, and the Ubuntu-specific post-install fixes (service conflicts, D-Bus policy, udev/kernel modules, group membership) required to get it actually working.

> **Note on kernel version:** TDP control depends on the `asus-armoury` kernel driver, mainline only since Linux 6.19. Ubuntu LTS releases ship much older kernels, so on stock Ubuntu 24.04 (and older) expect TDP sliders to be unavailable until you're on a newer kernel (e.g. via the mainline/liquorix/xanmod kernel PPAs). Everything else in asusctl works fine on older kernels.

1. Install System Dependencies

Ubuntu uses different package names than the Fedora defaults. This list merges what's been confirmed needed to build both `asusd` and the `rog-control-center` GUI on Ubuntu:

    sudo apt update
    sudo apt install -y libclang-dev libudev-dev libseccomp-dev \
    libseat-dev libsensors4-dev libpulse-dev libssl-dev libx11-dev \
    libxkbcommon-dev libgbm-dev libinput-dev libfontconfig1-dev \
    libfreetype6-dev libexpat1-dev libxcb-composite0-dev libasound2-dev \
    pkg-config build-essential git cmake

If you only need the CLI/daemon (`asusctl`/`asusd`) and not the GUI, you can drop the GTK/X11-related packages (`libx11-dev`, `libxkbcommon-dev`, `libgbm-dev`, `libinput-dev`, `libfontconfig1-dev`, `libfreetype6-dev`, `libxcb-composite0-dev`).

2. Install Rust Toolchain

The version of Rust in the Ubuntu repositories is often too old. Use rustup to get the latest stable version:

Install Rust:

    curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

Refresh your shell:

    source $HOME/.cargo/env

Set the Clang Path: Rust needs to know where libclang is. Note that the version number (e.g., 14) may vary depending on your Ubuntu version. Check /usr/lib/llvm-*/lib to verify.

    export LIBCLANG_PATH=/usr/lib/llvm-14/lib

3. Clone the Repository

    git clone https://gitlab.com/asus-linux/asusctl.git
    cd asusctl

> **No manual group patching needed anymore.** Older versions of this guide had you `sed` a `GROUP="wheel"` udev rule to `GROUP="sudo"`. As of current upstream, that string doesn't even exist in the udev rules file (`data/asusd.rules`) — the actual permission gate is the D-Bus policy in `data/asusd.conf`, and it already ships with `<policy group="sudo">` alongside `wheel`, `adm`, and `users`, so Ubuntu accounts (which are in `sudo`, not `wheel`) are covered out of the box. If you still hit a D-Bus permission error after install, see the Troubleshooting FAQ below — it's almost always the missing `/etc/dbus-1/system.d` symlink, not group ownership.

> Upstream's issue tracker and Arch packaging have moved to GitHub ([OpenGamingCollective/asusctl](https://github.com/OpenGamingCollective/asusctl)); the GitLab URL above still works as a mirror for cloning, but file bugs/questions against the GitHub repo.

4. Build and Install

Now, compile the project and install it to your system.

    make
    sudo make install

5. Critical Post-Install Steps

These steps are mandatory for the software to actually control your hardware.

A. Resolve Service Conflicts

Ubuntu's default power-profiles-daemon will conflict with asusd. You must disable it:

    sudo systemctl stop power-profiles-daemon
    sudo systemctl mask power-profiles-daemon

B. Enable Kernel Modules (RGB/Matrix Support)

If your laptop has RGB lighting or an AniMe Matrix display, you must load the i2c-dev module:
Bash

# Load for the current session
    sudo modprobe i2c-dev

# Ensure it loads on every boot
    echo "i2c-dev" | sudo tee /etc/modules-load.d/i2c-dev.conf

C. Update User Groups

Add your user to the necessary groups to allow communication with the hardware without constant sudo prompts:

    sudo usermod -aG video,input,i2c $USER

Note: You must log out and log back in for these group changes to take effect.
6. Activation

Reload the systemd manager and start the daemon:

    sudo systemctl daemon-reload
    sudo systemctl enable --now asusd

If you installed the graphical interface (rog-control-center), update the desktop database so it appears in your app drawer:

    sudo update-desktop-database

Troubleshooting FAQ

D-Bus Policy Error: If the service fails to start, Ubuntu may need a symlink for the D-Bus configuration: sudo ln -s /usr/share/dbus-1/system.d/asusd.conf /etc/dbus-1/system.d/asusd.conf

Brightness Keys: If your screen brightness keys stop working, add acpi_backlight=vendor to your Grub config:

    sudo nano /etc/default/grub

Add the parameter to GRUB_CMDLINE_LINUX_DEFAULT.
Run

    sudo update-grub and reboot.

**Note:** on at least one tested OLED panel (GA403UI), `acpi_backlight=vendor`/`video` and `amdgpu.backlight=0` had **no effect** on brightness at all — the panel's ACPI backlight interface accepts writes but doesn't actually change hardware brightness. If this GRUB param doesn't fix it for you, the real cause is likely an unstable `amdgpu_bl*` device name across boots, not the ACPI backlight path — see [`TESTED-CONFIG-GA403UI.md`](TESTED-CONFIG-GA403UI.md#1-brightness-keys-stop-working-intermittently) for a fix that resolves that case.
