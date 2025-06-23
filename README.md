# Yocto Build System for Raspberry Pi 4 with Docker, Systemd, and Camera Support

## Overview

This project sets up a custom Linux image for the **Raspberry Pi 4** using the **Yocto Project** and includes layers for virtualization, media support, and development utilities. It targets **systemd**, **Docker**, and **libcamera** integration, with support for various hardware features like **V4L2 cameras (OV5647, bcm2835-unicam)**.

We are using **Yocto Mickledore** release, as `rpi-libcamera-apps` requires **meson-native > 0.60** which is fulfilled in this release.

---

## Layer Setup and Purpose

Layers are specified in `bblayers.conf`. Here's a breakdown of the key layers:

| Layer                                | Purpose |
|-------------------------------------|---------|
| `meta` / `meta-poky`                | Core OpenEmbedded and Yocto metadata |
| `meta-raspberrypi`                  | Raspberry Pi-specific board support and configurations |
| `meta-openembedded/meta-oe`         | Additional general-purpose recipes |
| `meta-openembedded/meta-python`     | Python and related libraries support |
| `meta-openembedded/meta-networking` | Networking utilities and libraries |
| `meta-openembedded/meta-filesystems`| File system tools and packages |
| `meta-openembedded/meta-multimedia` | Multimedia support (codecs, players) |
| `meta-virtualization`               | Virtualization support including Docker and QEMU |
  
---

## Required Layers and Download Instructions

Clone the required layers into your `sources` directory:

```bash
cd yocto/sources/

# Poky (includes meta and meta-poky)
git clone -b mickledore git://git.yoctoproject.org/poky

# Meta Raspberry Pi
git clone -b mickledore https://github.com/agherzan/meta-raspberrypi.git

# Meta OpenEmbedded
git clone -b mickledore https://github.com/openembedded/meta-openembedded.git

# Meta Virtualization
git clone -b mickledore https://git.yoctoproject.org/git/meta-virtualization
```

Ensure all branches are aligned to mickledore, especially for meta-raspberrypi and meta-openembedded.
Build Initialization

To initialize and build the image:

```bash
cd yocto
source sources/poky/oe-init-build-env
```

# Then build the minimal image

```bash
bitbake core-image-minimal
```

Full Command-Line Image (Recommended)

We use a custom image called core-image-full-cmdline, which extends core-image-minimal and includes:

    Python3, Git

    Docker (docker-moby)

    OpenSSH server

    Networking tools: curl, vim, nano, ifmetric, htop, tmux

    Systemd as init manager

    Camera support: libcamera, v4l-utils, rpi-libcamera-apps, i2c-tools

    Kernel modules for ov5647, bcm2835-unicam, and general media API drivers

To build it:

```bash
bitbake core-image-full-cmdline
```

Ensure your local.conf contains all required settings (already configured in this setup).
Device and Peripheral Support
Enabled via local.conf:

    UART

    I2C

    SPI

    DT overlays

    GPU Memory set to 128

    Camera with START_X = "1"

V4L2 Media Support:

We explicitly enable Video4Linux2 drivers and kernel modules for:

    ov5647

    bcm2835-unicam

    Media controller APIs

Docker Support

Docker is enabled via:

bash```
DISTRO_FEATURES:append = " virtualization"
PREFERRED_PROVIDER_virtual/docker = "docker-moby"
IMAGE_INSTALL:append = " docker-moby"
```

This allows containerized development directly on the Raspberry Pi.
Systemd Configuration

Systemd is enabled as the primary init system:

bash```
DISTRO_FEATURES:append = " systemd udev"
VIRTUAL-RUNTIME_init_manager = "systemd"
VIRTUAL-RUNTIME_dev_manager = "udev"
```

This disables sysvinit and ensures systemd services behave as expected.
Menuconfig and Diffconfig

You can customize the kernel using:

bash```
bitbake virtual/kernel -c menuconfig
```

This launches a terminal-based kernel configuration interface.

After making changes, save them and store your diff:

bash```
bitbake virtual/kernel -c diffconfig
```

This outputs your changes so they can be committed or reused as patches.
Notes

    We use mickledore for compatibility with rpi-libcamera-apps, which requires meson-native > 0.60.

    rpi-libcamera-apps provides camera support for the Raspberry Pi using the modern libcamera stack.

    Kernel modules for camera support must be explicitly included in IMAGE_INSTALL.

Final Build Summary

To summarize:

bash```
cd yocto
source sources/poky/oe-init-build-env
bitbake core-image-full-cmdline
```

Generated images can be found in:

bash```
build/tmp/deploy/images/raspberrypi4/
```

Flash the .wic.bz2 or .img file to an SD card using tools like balenaEtcher or dd.
Optional: Save Disk Space

Preserve downloads and shared-state cache across builds:

