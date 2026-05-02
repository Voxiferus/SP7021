# SP7021 uEnv Examples

These files are reference `uEnv.txt` variants for the SD-card FAT boot
partition.

## SD Rootfs

`uEnv.sd-rootfs.txt` is the SDK default flow:

```text
kernel from SD FAT partition
DTB from U-Boot fdtcontroladdr
root filesystem from /dev/mmcblk1p2
```

## TFTP Kernel and NFS Rootfs

`uEnv.tftp-nfs.txt` is the tested development flow:

```text
bootloader from SD card
kernel and DTB over TFTP
root filesystem over NFS
```

Before using it, adjust these variables if your network or NFS path is
different:

```text
SERVERIP=192.168.1.78
CLIENTIP=192.168.1.199
nfsroot=${SERVERIP}:/home/leo/sunplus/nfs/sp7021
```

Copy the selected file to the generated SD output as `uEnv.txt` before copying
it to the SD-card FAT partition:

```sh
cp board/sp7021/uenv/uEnv.tftp-nfs.txt out/boot2linux_SDcard/uEnv.txt
```
