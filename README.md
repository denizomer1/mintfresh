# OxiKernel

OxiKernel is an AOSP-only Android kernel built exclusively for the **Samsung Galaxy A50** (SM-A505x). It targets AOSP, GSI, and LineageOS-based systems,
> [!WARNING]
> Flashing a custom kernel can cause bootloops, data loss, or permanent security feature disablement. Back up your data before proceeding. You are solely responsible for any issues that may arise.

## Overview

| Feature | Details |
| --- | --- |
| Supported device | Samsung Galaxy A50 (`a50`, SM-A505 family) |
| SoC | Samsung Exynos 9610 |
| Kernel base | Linux 4.14.336 |
| Supported Android version | Android 11 - Android 16 |
| ROM types | AOSP-based ROMs only |
| Architectures | ARM64 kernel, ARM32 compatibility layer |
| Compiler | Proton Clang 13 (denizomer1/proton-clang) |
| License | Kernel: GPL-2.0, build script: GPL-3.0 |

### Key features

- Built with Clang and Link-Time Optimization (LTO)
- Low-latency optimized Exynos Mobile Scheduler (EMS)
- Galaxy S10/S20-inspired scheduler and governor enhancements
- Additional I/O schedulers; `anxiety` is the default
- ZRAM with LZ4 compression for effective RAM expansion
- Per-process swap support for better memory management
- WireGuard VPN support
- Custom configuration for AOSP-based ROMs
- Upstream Linux and Android device ported fixes

The device-tree section includes only A50 SWA hardware revisions (`00`, `01`, `02`, `03`, `04`, `06`) and their Exynos 9610 dependencies. DTS files for the A50s, A80, M30s, Exynos 9611, and other ARM64 boards have been removed. Samsung display, camera, modem, audio, sensor, and power drivers required for hardware functionality have been preserved.

## Requirements

- Unlocked bootloader
- TWRP, SHRP, or any compatible custom recovery that can flash ZIP files
- The correct OxiKernel package for your device and ROM
- Backup of important data and the current `boot` partition

### Important notes

- Samsung Knox counter may be permanently triggered when the bootloader is unlocked. Banking apps, Secure Folder, and some DRM features may be affected.
- OxiKernel is built only for LineageOS and other AOSP-based ROMs. Samsung stock firmware and One UI-based ROMs are not supported.
- `Enforcing` is recommended for normal use. `Permissive` is for debugging only and is not secure.

## Installation

1. Download the appropriate OxiKernel ZIP from the `out/` directory.
2. Copy the package to your phone's internal storage or SD card.
3. Reboot into recovery mode.
4. Back up the existing `boot` partition if possible.
5. Flash the OxiKernel ZIP.
6. Reboot directly into the system instead of recovery.

The first boot may take longer than usual. If the device does not boot, return to recovery and restore your backed-up `boot` image or the ROM's original kernel.

## Building from source

Building is supported only on **64-bit Linux**. Ubuntu 22.04 LTS is the smoothest environment and matches CI. Current Ubuntu/Debian, Fedora, and Arch-based distributions can also build with the dependencies below; due to older Proton Clang binaries, some very new distributions may require extra compatibility packages.

At least **20 GB** of free disk space and **8 GB** of RAM are recommended. Internet access is required on the first build to download the toolchain.

### 1. Install dependencies

#### Ubuntu 22.04/24.04 and Debian 12/13

```bash
sudo apt update
sudo apt install -y \
  bc binutils bison build-essential bzip2 cpio flex git jq \
  libelf-dev libncurses-dev libssl-dev libc6-dev-i386 lib32stdc++6 \
  python3 rsync unzip zip
```

#### Fedora

```bash
sudo dnf install -y \
  @development-tools bc binutils bison bzip2 cpio elfutils-libelf-devel \
  flex git glibc-devel.i686 jq ncurses-devel openssl-devel python3 \
  rsync unzip zip
```

#### Arch Linux and derivatives

```bash
sudo pacman -S --needed \
  base-devel bc binutils bison bzip2 cpio flex git jq libelf ncurses \
  openssl python rsync unzip zip
```

### 2. Clone the repository

```bash
git clone https://github.com/denizomer1/oxikernel.git
cd oxikernel
```

### 3. Start the build

`build.sh` automatically downloads the required Proton Clang toolchain from `denizomer1/proton-clang` into `toolchain/` on the first run, merges configurations, and produces the package.

For SELinux enforcing build:

```bash
./build.sh --enforcing
```

For SELinux permissive build:

```bash
./build.sh --permissive
```

### Build options

```text
--enforcing     Produces kernel with SELinux enforcing.
--permissive    Produces kernel with SELinux permissive (not recommended).
```

OxiKernel is pinned to Android 16 GSI. There are no options for other Android versions, devices, ROM variants, or incremental builds. The script always performs a clean build.

### Build outputs

The build artifacts are written to the `out/` directory:

- `OxiKernel-*.zip`: Minimal package flashable via Galaxy A50 recovery
- `boot.img`: Direct boot image for Galaxy A50

The ZIP contains only `boot.img`, `dtb.img`, and a minimal recovery installer specific to the Galaxy A50. It does not include AnyKernel, BusyBox, Magisk, KernelSU, or any root helper tool. The installer checks the device ID only and writes the two images to the corresponding partitions.

To rebuild from scratch, rerun the normal command. The script always creates a clean configuration with `make mrproper`.

### Using with LineageOS/GSI development

This tree can be used as a standard ARM64 kernel source inside the A50 device tree. The build configuration consists of three parts:

- `arch/arm64/configs/exynos9610-a50_core_defconfig`
- `arch/arm64/configs/exynos9610-a50_default_defconfig`
- `kernel/configs/oxikernel_*12.config`

If the ROM build system packages the kernel itself, it must use the same config merge order as in `build.sh`. `build.sh` serves as a standalone reference application for testing and recovery ZIP production.

## Troubleshooting

### Toolchain cannot be downloaded

`git` must be installed and GitLab must be accessible. If there is a partial download, remove the `toolchain/` directory and rerun the build.

### `libncurses.so.5` not found

Proton Clang 13 targets an older userspace ABI, so this error may appear on some newer distributions. First use your distribution's compatibility package. If the package is not available, build inside an Ubuntu 22.04-based container or virtual machine; do not create manual symlinks in system library paths.

### `Image not built successfully` error after build

This message summarizes the actual error. Inspect the first `error:` line that appears before it in the terminal output.

### Device does not boot

Verify that the package matches your device, ROM type, and Android version. Then restore the previous `boot` backup or the ROM's original boot image from recovery.

## Bug report

When reporting a bug, include the following:

- Exact device model (e.g. `SM-A505F`)
- OxiKernel version and package name used
- Android and ROM version
- Baseband version
- Steps to reproduce the issue
- Relevant `logcat`, `dmesg`, and `kmsg` logs

Bugs can be reported at [GitHub Issues](https://github.com/denizomer1/oxikernel/issues).
