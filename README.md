# asusctl-for-Ubuntu
Porting asusctl (made for Fedora) for Ubuntu

Installing asusctl on Ubuntu

asusctl is a fantastic utility for ASUS ROG/TUF laptops, but it is primarily packaged for Fedora and Arch. This guide provides the specific steps, manual patches, and troubleshooting fixes required to get it running perfectly on Ubuntu.
1. Install System Dependencies

Ubuntu uses different package names than the Fedora defaults. Run this to prepare your environment:

    sudo apt update
    sudo apt install -y libclang-dev libudev-dev libseccomp-dev \
    libseat-dev libsensors4-dev libpulse-dev \
    build-essential git cmake

2. Install Rust Toolchain

The version of Rust in the Ubuntu repositories is often too old. Use rustup to get the latest stable version:

Install Rust:

    curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

Refresh your shell:

    source $HOME/.cargo/env

Set the Clang Path: Rust needs to know where libclang is. Note that the version number (e.g., 14) may vary depending on your Ubuntu version. Check /usr/lib/llvm-*/lib to verify.

    export LIBCLANG_PATH=/usr/lib/llvm-14/lib

3. Clone and Manual Patching

Since asusctl expects Fedora's group permissions, we need to patch the source before building.

Clone the repo:

    git clone https://gitlab.com/asus-linux/asusctl.git
    cd asusctl

Fix Group Permissions (udev): Ubuntu uses the sudo group for administrative privileges, while Fedora uses wheel. Run this command to swap them in the configuration:

    sed -i 's/GROUP="wheel"/GROUP="sudo"/g' data/99-asusd.rules

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
