# SP7021 SDK Build Notes

This repository builds firmware for Sunplus SP7021-based boards. The tested
development target is `LTPP3G2 Board (S+)`, which enables dual-NIC mode in the
SP7021 L2 switch device tree.

The current flow uses the vendor SDK for xBoot, U-Boot, Linux kernel and DTB,
and Buildroot 2026.02 for the root filesystem.

## Host Packages

Install the host build dependencies:

```sh
sudo apt-get install openssl libssl-dev bison flex git make u-boot-tools libmpc-dev libgmp-dev python3-pip mtd-utils libncurses* mtools
pip install pycryptodomex pyelftools Crypto
```

## Source Checkout

Fetch the source code:

```sh
git clone https://github.com/Voxiferus/SP7021.git
cd SP7021
git submodule update --init --recursive
```
Do not run `git submodule foreach git checkout master` for a normal build.
The top-level repository pins each submodule to an exact commit, which is what
makes the SDK tree reproducible. Checking out `master` inside every submodule
moves them away from the pinned commits and leaves the top-level repository
dirty. If this happened accidentally, restore the pinned submodule commits with:

```sh
git submodule update --init --recursive --checkout
```

## SDK Configuration

Configure the build:

```sh
make config
```

Select your board:

```text
Select boards:
[1] SP7021 Ev Board     [11] I143 Ev Board      [21] Q645 Ev Board      [31] SP7350 Ev Board
[2] LTPP3G2 Board       [12] I143 Zebu (ZMem)   [22] Q645 Zebu (ZMem)   [32] SP7350 Zebu (ZMem)
[3] SP7021 Demo Brd V2
[4] SP7021 Demo Brd V3
[5] BPI-F2S Board
[6] BPI-F2P Board
[7] LTPP3G2 Board (S+)
```
If you selected [1] or [11], please select the chip:

```text
Select chip.
[1] Chip C
[2] Chip P
```

```text
Select configs (C chip).
[1] eMMC
[2] SD Card
```

```text
Select rootfs:
[1] BusyBox
[2] Yocto
[3] Ubuntu Server 20.04
[4] Buildroot
```
`Buildroot` is available for `[2] LTPP3G2 Board` and `[7] LTPP3G2 Board (S+)`.
It keeps the vendor SDK responsible for xBoot, U-Boot, the Linux kernel and the
DTB, while using an external Buildroot tree for the root filesystem.

For the tested board, use:

```text
Board:  [7] LTPP3G2 Board (S+)
Boot:   [2] SD Card
Rootfs: [4] Buildroot
```

## Build

Build the selected SDK configuration:

```sh
make
```

If your local `LANG` is not English, run:

```sh
LANG=c make
```

When `Rootfs: [4] Buildroot` is selected, `make` automatically downloads and
extracts Buildroot 2026.02, applies `buildroot-sp7021-nfs.defconfig`, builds
`buildroot-output/images/rootfs.tar`, and converts it to `out/rootfs.img`.

Important output files:

```text
out/uImage
out/dtb
out/rootfs.img
linux/rootfs/rootfs.img
out/boot2linux_SDcard/ISP_SD_BOOOT.img
```

## Buildroot

Internally, the SDK runs the equivalent of:

```sh
make -C buildroot-2026.02 O="$PWD/buildroot-output" BR2_DEFCONFIG="$PWD/buildroot-sp7021-nfs.defconfig" defconfig
make -C buildroot-2026.02 O="$PWD/buildroot-output" BR2_DL_DIR="$PWD/buildroot-dl"
```

The generated Buildroot artifact is:

```text
buildroot-output/images/rootfs.tar
```

The SDK `make rootfs` target converts that tarball into:

```text
out/rootfs.img
linux/rootfs/rootfs.img
```

The SD-card image generator then uses `out/rootfs.img` as the second partition
when `ROOTFS_CONTENT=BUILDROOT`.

To use a non-default Buildroot output path:

```sh
make rootfs BUILDROOT_ROOTFS_TAR=/path/to/rootfs.tar
```

To override the Buildroot version or source URL:

```sh
make rootfs BUILDROOT_VERSION=2026.02 BUILDROOT_URL=https://buildroot.org/downloads/buildroot-2026.02.tar.xz
```

## Reconfiguring Buildroot

Apply the saved Buildroot defconfig before editing it:

```sh
make buildroot-defconfig
```

Open Buildroot menuconfig:

```sh
make -C buildroot-2026.02 O="$PWD/buildroot-output" menuconfig
```

Save the edited minimal defconfig back to the tracked SDK file:

```sh
make -C buildroot-2026.02 O="$PWD/buildroot-output" BR2_DEFCONFIG="$PWD/buildroot-sp7021-nfs.defconfig" savedefconfig
```

For an experimental configuration, save to a different file:

```sh
make -C buildroot-2026.02 O="$PWD/buildroot-output" BR2_DEFCONFIG="$PWD/buildroot-sp7021-nfs-experimental.defconfig" savedefconfig
```

To build with a non-default saved Buildroot configuration, pass it to the SDK:

```sh
make BUILDROOT_DEFCONFIG="$PWD/buildroot-sp7021-nfs-experimental.defconfig"
```

The default SDK build uses:

```text
buildroot-sp7021-nfs.defconfig
```

## Reconfiguring the Kernel

Run `make config` first so the SDK selects the board-specific kernel defconfig.
Then open the kernel menuconfig through the SDK:

```sh
make kconfig
```

Do not run `make menuconfig` directly inside `linux/kernel`; the SDK provides
the required architecture and cross-compiler settings.

The edited kernel configuration is stored in:

```text
linux/kernel/.config
```

To keep a local full kernel configuration snapshot:

```sh
mkdir -p board/sp7021/kernel-configs
cp linux/kernel/.config board/sp7021/kernel-configs/sp7021-ltpp3g2-sunplus.config
```

To apply that saved kernel configuration later:

```sh
cp board/sp7021/kernel-configs/sp7021-ltpp3g2-sunplus.config linux/kernel/.config
```

Kernel configuration snapshots are not currently wired into the SDK build
automation; keep them as explicit project files if you need reproducible kernel
configuration variants.

## SSH Key

The committed Buildroot overlay intentionally contains an empty file:

```text
board/sp7021/rootfs-overlay/root/.ssh/authorized_keys
```

Do not commit personal SSH keys. Before building a private image, copy your
host public SSH key into that file locally:

```sh
cp ~/.ssh/<public-key>.pub board/sp7021/rootfs-overlay/root/.ssh/authorized_keys
```

Buildroot applies reproducible permissions through:

```text
board/sp7021/device_table.txt
```

which sets:

```text
/root                       0755 root root
/root/.ssh                  0700 root root
/root/.ssh/authorized_keys  0600 root root
```

When the build completes, you will find the image file in the `out` folder.

If you chose
* eMMC:
  * Copy `out/ISPBOOOT.BIN` to a USB stick, which should be a FAT32 filesystem
  * Update the eMMC from the USB stick:  power off, set SW1 to on, SW2 to off, power on
  * Boot the kernel: power off, set SW1 to off, SW2 to off, power on
* SD Card:
  * the image file is `out/boot2linux_SDcard/ISP_SD_BOOOT.img`
  * write it to the sdcard: `sudo dd if=ISP_SD_BOOOT.img of=/dev/sdX bs=1M`, where /dev/sdX is your sdcard device
  * insert your sdcard in the board and boot: power off, set SW1 to on, SW2 to on, power on

On the tested board, SD boot requires shorting jumpers:

```text
CN01 and CN11
```

Without those jumpers, the board boots from eMMC and ignores the SD card.

## TFTP and NFS Development Boot

The tested development flow is:

```text
bootloader from SD card
kernel and DTB over TFTP
root filesystem over NFS
```

The tested wired server IP was:

```text
192.168.1.78
```

TFTP over Wi-Fi was unreliable. Use wired Ethernet for the TFTP server.

The tested `uEnv.txt` variants are stored in:

```text
board/sp7021/uenv/
```

For the TFTP/NFS development boot, copy the tested variant to the generated SD
output before copying it to the SD-card FAT partition:

```sh
cp board/sp7021/uenv/uEnv.tftp-nfs.txt out/boot2linux_SDcard/uEnv.txt
```

## Ethernet Notes

In dual-NIC mode Linux registers:

```text
eth0
eth1
```

The driver uses one shared SP7021 L2 switch/MAC block and one shared IRQ for
both Linux netdevs. Keep management/NFS/SSH traffic on `eth0` and dedicate
`eth1` to EtherCAT when possible.

Before running a SOEM-based EtherCAT master on `eth1`, bring the interface up:

```sh
ip addr flush dev eth1
ip link set eth1 up
```

For more information, please visit [here](https://sunplus-tibbo.atlassian.net/wiki/spaces/doc/pages/375783435/SP7021+Application+Note)
