---
layout: post
title:  "Yocto on Beaglebone Black"
date:   2020-12-16 23:26:00
categories: wireless, linux, programming
---

Below are the steps to build and flash Yocto on an emmc.


## Build

```bash
mkdir yocto-bbb/
cd yocto-bbb/
git clone -b thud git://git.yoctoproject.org/poky.git
source poky/oe-init-build-env build
echo 'MACHINE = "beaglebone-yocto"' > conf/local.conf
echo 'EXTRA_IMAGE_FEATURES += " debug-tweaks "' >> conf/local.conf
bitbake core-image-minimal
```

After a successful build, the images should be in `tmp-glibc/deploy/images/beaglebone-yocto/`. Something like the below,

```bash
am335x-bone-beaglebone-yocto.dtb                                                             core-image-minimal-beaglebone-yocto.tar.bz2
am335x-boneblack--4.18.35+git0+865683fc87_b24d9d2146-r0-beaglebone-yocto-20201216104117.dtb  core-image-minimal-beaglebone-yocto.testdata.json
am335x-boneblack-beaglebone-yocto.dtb                                                        core-image-minimal-beaglebone-yocto.wic
am335x-boneblack.dtb                                                                         core-image-minimal-beaglebone-yocto.wic.bmap
am335x-bone.dtb                                                                              MLO
am335x-bonegreen--4.18.35+git0+865683fc87_b24d9d2146-r0-beaglebone-yocto-20201216104117.dtb  MLO-beaglebone-yocto
am335x-bonegreen-beaglebone-yocto.dtb                                                        MLO-beaglebone-yocto-2018.07-r0
am335x-bonegreen.dtb                                                                         modules--4.18.35+git0+865683fc87_b24d9d2146-r0-beaglebone-yocto-20201216104117.tgz
core-image-minimal-beaglebone-yocto-20201216181833.rootfs.jffs2                              modules-beaglebone-yocto.tgz
core-image-minimal-beaglebone-yocto-20201216181833.rootfs.manifest                           u-boot-beaglebone-yocto-2018.07-r0.img
core-image-minimal-beaglebone-yocto-20201216181833.rootfs.tar.bz2                            u-boot-beaglebone-yocto.img
core-image-minimal-beaglebone-yocto-20201216181833.rootfs.wic                                u-boot.img
core-image-minimal-beaglebone-yocto-20201216181833.rootfs.wic.bmap                           zImage
core-image-minimal-beaglebone-yocto-20201216181833.testdata.json                             zImage--4.18.35+git0+865683fc87_b24d9d2146-r0-beaglebone-yocto-20201216104117.bin
core-image-minimal-beaglebone-yocto.jffs2                                                    zImage-beaglebone-yocto.bin

```

## Prepping the SDCARD

What we need is MLO, U-boot and the rootfs.tar.gz2 files to be on the SDCARD.

make two partitions of SDCARD on laptop.

```bash
Partition1 : Size 100 MB, fat16 (boot flag)
Partition2: Size > 2GB , ext4

```

The bootflag will appear on the corner right of the partition in Gparted.

Now, we need to copy MLO, u-boot and rootfs.

```bash
sudo cp MLO-beaglebone-yocto-2018.07-r0 /media/<user>/<partition1>/MLO
sudo cp u-boot-beaglebone-yocto-2018.07-r0.img /media/<user>/<partition1>/u-boot.img
sudo tar -xf core-image-minimal-beaglebone-yocto-20201216181833.rootfs.tar.bz2  -C /media/<user>/<partition2>/
sync

```

once done ,remove the SDCARD and put it in the sd slot of the Beaglebone Black.


## Serial port:

There is J1 header with 6 pins available on the Beaglebone Black. From the Ethernet jack side, count it from 1. 

1. pin 1 - black wire on FTDI cable - GND
2. pin 4 - green wire on FTDI cable - RX
3. pin 5 - white wire on FTDI cable - TX

use `minicom` Set baud 115200, select `/dev/ttyUSB0`.

## Boot with SD:

there is a button beside the slot area, called "S2". Hold it while the device is being plugged in to power. Holding S2 wil make the boot from the SDCARD.

There should be noticeable difference when booting from SDCARD and from eMMC. Check first few lines on the serial console during the boot, something like below.

```bash

U-Boot SPL 2018.07 (Dec 16 2020 - 12:29:03 +0000)
Trying to boot from MMC1
Loading Environment from FAT... *** Warning - bad CRC, using default environment

Failed (-5)
Loading Environment from MMC... *** Warning - bad CRC, using default environment

Failed (-5)


U-Boot 2018.07 (Dec 16 2020 - 12:29:03 +0000)

CPU  : AM335X-GP rev 2.1
I2C:   ready
DRAM:  512 MiB
No match for driver 'omap_hsmmc'
No match for driver 'omap_hsmmc'
Some drivers were not found
MMC:   OMAP SD/MMC: 0, OMAP SD/MMC: 1
Loading Environment from FAT... *** Warning - bad CRC, using default environment

Failed (-5)
Loading Environment from MMC... *** Warning - bad CRC, using default environment

Failed (-5)
<ethaddr> not set. Validating first E-fuse MAC
Net:   cpsw, usb_ether
Press SPACE to abort autoboot in 2 seconds

```

and after few lines the console should be visible.

```bash
EXT4-fs (mmcblk0p2): re-mounted. Opts: (null)
urandom_read: 3 callbacks suppressed
random: dd: uninitialized urandom read (512 bytes read)
INIT: Entering runlevel: 5
Configuring network interfaces... net eth0: initializing cpsw version 1.12 (0)
SMSC LAN8710/LAN8720 4a101000.mdio:00: attached PHY driver [SMSC LAN8710/LAN8720] (mii_bus:phy_addr=4a101000.mdio:00, irq=POLL)
IPv6: ADDRCONF(NETDEV_UP): eth0: link is not ready
udhcpc: started, v1.29.3
udhcpc: sending discover
udhcpc: sending discover
udhcpc: sending discover
udhcpc: no lease, forking to background
done.
Starting syslogd/klogd: done

OpenEmbedded nodistro.0 beaglebone-yocto /dev/ttyO0

beaglebone-yocto login: 
OpenEmbedded nodistro.0 beaglebone-yocto /dev/ttyO0

beaglebone-yocto login: root
root@beaglebone-yocto:~# 

```
