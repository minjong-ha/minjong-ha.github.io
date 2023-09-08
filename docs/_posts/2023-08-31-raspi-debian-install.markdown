---
layout: posts
title:  "Install Debian OS on raspberry pi with PREEMPT\_RT patch kernel"
author: Minjong Ha
published: true
date:   2023-08-31 15:06:22 +0900
---

This post introduces how to install Debian OS on raspberry pi and replace its kernel to PREEMPT\_RT patched kernel.

## Introduction

I began to conduct research related to the development of operating systems required for embedded systems.
For this purpose, I carried out research related to the development of a Linux operating system that can support real-time, and ultimately discovered that the PREEMPT\_RT patch kernel could be utilized.
Initially, I attempted to replace only the PREEMPT\_RT patch kernel on a Raspberry Pi with Raspbian OS installed, but due to the unfamiliar boot directory structure, I failed. 
Thus, I decided to install the Debian operating system on the Raspberry Pi and then replace the kernel.

The Raspberry Pi used for the setup is the `Raspberry Pi 3B+` model.


## Install Debian on Raspberry

Thanks for "Gunnar Wolf", who is the Debian Developer, I can download [compressed xz Debian 12 bookworm image](https://raspi.debian.net/tested-images/) for raspberry pi 3b+.
He manages [raspi.debian.net](https://raspi.debian.net) to support Debian OS for raspberry pi (it is not official Debian project).

You can install `compressed-xz` image on SD card with [raspberry pi imager](https://www.raspberrypi.com/software/).
Once you write image on SD card, just insert it on raspberry pi and boot.
You can see minimum configured Debian 12 running on raspberry pi.


## Cross-Compile Linux Kernel

Compiling linux source codes on embedded machine is painful due to its performance and resources.
I compiled linux kernel for raspberry pi on another Debian machine having x86\_64 intel processor.

### Prepare Linux Kernel and PREEMPT\_RT patch

```bash
wget https://mirrors.edge.kernel.org/pub/linux/kernel/v6.x/linux-6.1.46.tar.gz
wget https://cdn.kernel.org/pub/linux/kernel/projects/rt/6.1/patch-6.1.46-rt14-rc1.patch.gz

tar -xzf linux-xxxx.tar.gz
xz -d patch-xxxx.xz
```

### Collect Kernel Config and Installed Modules

```bash
# On raspberry pi 3B+
scp /boot/config-xxxx $(MACHINE_FOR_COMPILE)

lsmod > raspi_modules
scp raspi_modules $(MACHINE_FOR_COMPILE)
```

### Compile (Mainline)

```bash
# On Debian Desktop
mkdir mainline-6.1.46
cd linux-6.1.46/

# Configuration
cp ../config-xxxx .config
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- menuconfig
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- LSMOD=../raspi_modules localmodconfig

# Compile
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- -j$(nproc)
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- -j$(nproc) modules

#
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- -j$(nproc) INSTALL_PATH=../mainline-6.1.46 install
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- -j$(nproc) INSTALL_MOD_PATH=/mainline-6.1.46 modules_install

# Transfer mainline-6.1.46
ls mainline-6.1.46
System.map-6.1.46  config-6.1.46  lib  vmlinuz-6.1.46

scp -r mainline-6.1.46 pi@192.168.10.104:/home/pi/
```

I extract all kernel components in `mainline-6.1.46`. 
You should remove `source` and `build` symbolic link file in `mainline-6.1.46/lib/modules/6.1.46/` since it links linux source codes.

### Compile (PREEMPT\_RT)

```bash
# On Debian Desktop
mkdir rt-6.1.46
cd linux-6.1.46/

# Apply patch
patch -p1 < ../patch-6.1.46-rt14-rc1.patch

# Configuration
cp ../config-xxxx .config
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- menuconfig

# Enable CONFIG_PREEMPT_RT
 -> General Setup
  -> Preemption Model (Fully Preemptible Kernel (Real-Time))
   (X) Fully Preemptible Kernel (Real-Time) 

make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- LSMOD=../raspi_modules localmodconfig

# Compile
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- -j$(nproc)
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- -j$(nproc) modules

# 
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- -j$(nproc) INSTALL_PATH=../rt-6.1.46 install
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- -j$(nproc) INSTALL_MOD_PATH=/rt-6.1.46 modules_install

# Transfer mainline-6.1.46
ls rt-6.1.46
System.map-6.1.46-rt14-rc1  config-6.1.46-rt14-rc1  lib  vmlinuz-6.1.46-rt14-rc1

scp -r rt-6.1.46 pi@192.168.10.104:/home/pi/
``

Unlike compiling mainline kernel, `PREEMPT_RT` requires patch and modification for kernel config.
I only configured `CONFIG_PREEMPT_RT` to `Fully Premmptible Kernel (Real-Time)`, but there are some candidates that increase realtimeness.

```bash
# Enable CONFIG_HIGH_RES_TIMERS
 -> General setup
  -> Timers subsystem
   [*] High Resolution Timer Support

# Enable CONFIG_NO_HZ_FULL
 -> General setup
  -> Timers subsystem
   -> Timer tick handling (Full dynticks system (tickless))
    (X) Full dynticks system (tickless)

# Set CONFIG_HZ_1000 (note: this is no longer in the General Setup menu, go back twice)
 -> Processor type and features
  -> Timer frequency (1000 HZ)
   (X) 1000 HZ

# Set CPU_FREQ_DEFAULT_GOV_PERFORMANCE [=y]
 ->  Power management and ACPI options
  -> CPU Frequency scaling
   -> CPU Frequency scaling (CPU_FREQ [=y])
    -> Default CPUFreq governor (<choice> [=y])
     (X) performance
```

Above configurations represents candidates for realtimeness and I have plan to benchmark their differences.

### Install Kernel on Raspberry Pi

```
# On Raspberry pi 3B+
cp System.map-xxx /boot/
cp config-6.1.46 /boot/
cp vmlinuz-6.1.46 /boot/
cp lib/modules/6.1.46 /lib/modules/

# create initrd.img
sudo mkinitramfs -o /boot/initrd.img-6.1.46 6.1.46

# mv initrd and vmlinuz
sudo cp vmlinuz-xxxx /boot/firmware/
sudo cp initrd-xxxx /boot/firmware/

# Edit config.txt
kernel=vmlinuz-xxxx
initramfs initrd.img-xxxx

sudo reboot
```

Debian OS installed in raspberry pi uses its default bootloader.
Above commands represent how to install kernel components on raspberry pi.
There are five components: `System.map-xxx`, `config-xxxx`, `vmlinuz-xxxx`, `initrd.img-xxx`, and `/lib/modules/6.1.46`.
First, you should copy basic components on `/boot/` directory and create `initrd.img` file using `mkinitfamfs`.
Then replace `vmlinuz` and `initrd` in `/boot/firmware/` with modifying `config.txt`.

After the configurations, you can see replaced kernel after reboot.
