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

## Then build the minimal image

```bash
bitbake core-image-minimal
```

## Full Command-Line Image (Recommended)

We use a custom image called core-image-full-cmdline, which extends core-image-minimal and includes:

    Python3, Git

    Docker (docker-moby)

    OpenSSH server

    Networking tools: curl, vim, nano, ifmetric, htop, tmux

    Systemd as init manager

    Camera support: libcamera, v4l-utils, rpi-libcamera-apps, i2c-tools

    Kernel modules for ov5647, bcm2835-unicam, and general media API drivers

## To build it:

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

## V4L2 Media Support:

We explicitly enable Video4Linux2 drivers and kernel modules for:

    ov5647

    bcm2835-unicam

    Media controller APIs

## Docker Support

Docker is enabled via:

```bash
DISTRO_FEATURES:append = " virtualization"
PREFERRED_PROVIDER_virtual/docker = "docker-moby"
IMAGE_INSTALL:append = " docker-moby"
```

This allows containerized development directly on the Raspberry Pi.
Systemd Configuration

## Systemd is enabled as the primary init system:

```bash
DISTRO_FEATURES:append = " systemd udev"
VIRTUAL-RUNTIME_init_manager = "systemd"
VIRTUAL-RUNTIME_dev_manager = "udev"
```

This disables sysvinit and ensures systemd services behave as expected.
Menuconfig and Diffconfig

You can customize the kernel using:

```bash
bitbake virtual/kernel -c menuconfig
```

This launches a terminal-based kernel configuration interface.

After making changes, save them and store your diff:

```bash
bitbake virtual/kernel -c diffconfig
```

This outputs your changes so they can be committed or reused as patches.

### Notes
We use mickledore for compatibility with rpi-libcamera-apps, which requires meson-native > 0.60.
rpi-libcamera-apps provides camera support for the Raspberry Pi using the modern libcamera stack.
Kernel modules for camera support must be explicitly included in IMAGE_INSTALL.

# Final Build Summary

To summarize:

```bash
cd yocto
source sources/poky/oe-init-build-env
bitbake core-image-full-cmdline
```

Generated images can be found in:

```bash
build/tmp/deploy/images/raspberrypi4/
```

## Flashing the Image and Resizing the SD Card

Once the image has been built using Yocto, you'll find output files in:

```bash
build/tmp/deploy/images/raspberrypi4/
```

### 1. Identify and Uncompress the Image

List available .wic image files (compressed or not):

```bash
ls -l *wic*
```

You will likely see files like:

```bash
-rw-r--r-- 1 user user  314M Jun 23 12:34 core-image-full-cmdline-raspberrypi4.wic.bz2
```

Optional: Copy the File First
In some cases, you will have to copy the image before unziping it:

```bash
cp core-image-full-cmdline-raspberrypi4.wic.bz2 ~/Downloads/
cd ~/Downloads/
```

Unzip the .wic.bz2 File

Use bzip2 to decompress the image:

```bash
bzip2 -d core-image-full-cmdline-raspberrypi4.wic.bz2
```

This will leave you with:

```bash
core-image-full-cmdline-raspberrypi4.wic
```
### 2. Flash to SD Card (Using Balena Etcher)

Use Balena Etcher (GUI or CLI) to flash the .wic image to your SD card.

    ⚠️ Make sure to select the correct target device. Flashing will overwrite all existing data on the SD card.

### 3. Resize the Partition After Flashing

By default, Yocto-generated .wic images leave unused space unallocated. To make full use of your SD card's storage:
a. Open fdisk on the SD card device

Replace /dev/sdb with your actual SD card device. Double-check with lsblk.

```bash
sudo fdisk /dev/sdb
```

b. In fdisk, follow this sequence:

```bash
Command (m for help): p
# Print the partition table

Command (m for help): d
Partition number (1,2,...): 2
# Delete the second (root) partition

Command (m for help): n
Partition type: primary
Partition number: 2
First sector: [Press ENTER to accept default]
Last sector: [Press ENTER to use all remaining space]
```

> 🛑 In some cases, `fdisk` may warn:
>
>     "The signature will be removed by a write command. Do you want to remove the signature? [Y]es/[N]o:"
>
> Choose **No** to keep the ext4 signature intact. This ensures the existing filesystem remains recognizable and resizable.

```bash
Command (m for help): w
# Write changes and exit
```

    🚫 Do not delete partition 1 (boot), and do not format the partition—just recreate it using the full space.

### 4. Resize the ext4 Filesystem Inside a Docker Container

If your host system is missing resize tools, use a temporary Docker container:

```bash
docker run --rm -it --privileged -v /dev:/dev ubuntu:24.04
```

Inside the container, run:

```bash
apt update && apt install -y e2fsprogs
e2fsck -f /dev/sdb2
resize2fs /dev/sdb2
```
This will safely expand the root filesystem to fill the resized partition.

    ⚠️ Ensure /dev/sdb2 is not mounted during this operation.

You're Done!

You can now boot your Raspberry Pi 4 with the resized and fully functional Yocto-based image.


# Clone the Robot Navigation Repository

Now that your Raspberry Pi has successfully booted into the Yocto-based operating system, you can proceed to prepare the robot navigation software.

## Clone the application repository:

```bash
git clone https://github.com/rayamech/robot-navigation
cd robot-navigation
```

## Switch to the Yocto branch:

```bash
git checkout yocto
```

At this point, follow the instructions provided in the repository to build and run the application.

