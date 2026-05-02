# SP7021 Buildroot Rootfs Flow

This repository uses the Sunplus SDK for xBoot, U-Boot, Linux kernel and DTB,
and an external Buildroot tree for the root filesystem.

The tested board is `LTPP3G2 Board (S+)`, which enables dual-NIC mode in the
SP7021 L2 switch device tree.

## SDK Configuration

Run:

```sh
make config
```

Use:

```text
Board:  [7] LTPP3G2 Board (S+)
Boot:   [2] SD Card
Rootfs: [4] Buildroot
```

For the non-Sunplus LTPP3G2 variant, `Board: [2] LTPP3G2 Board` also exposes
the Buildroot rootfs option.

## Buildroot

Use Buildroot 2026.02 or newer as a separate source tree:

```sh
make -C buildroot-2026.02 O="$PWD/buildroot-output" BR2_DEFCONFIG="$PWD/buildroot-sp7021-nfs.defconfig" defconfig
make -C buildroot-2026.02 O="$PWD/buildroot-output" BR2_DL_DIR="$PWD/buildroot-dl"
```

The SDK expects this artifact:

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

## SSH Key

The committed overlay intentionally contains an empty file:

```text
board/sp7021/rootfs-overlay/root/.ssh/authorized_keys
```

Do not commit personal SSH keys. Before building a private image, copy your
public key into that file locally:

```sh
cp ~/.ssh/id_ed25519.pub board/sp7021/rootfs-overlay/root/.ssh/authorized_keys
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

## SD Boot

On the tested board, SD boot requires shorting jumpers:

```text
CN01 and CN11
```

Without those jumpers, the board boots from eMMC and ignores the SD card.

Write the generated SD image to the whole card device:

```sh
sudo dd if=out/boot2linux_SDcard/ISP_SD_BOOOT.img of=/dev/sdX bs=4M status=progress conv=fsync
sync
```

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
