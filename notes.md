# notes

Exploring the strategy.

## 2024-06-29 boot

* <https://www.raspberrypi.com/documentation/computers/raspberry-pi.html#raspberry-pi-boot-eeprom>
* <https://www.raspberrypi.com/documentation/computers/raspberry-pi.html#eeprom-boot-flow>
* <https://www.raspberrypi.com/documentation/computers/raspberry-pi.html#raspberry-pi-boot-modes>

In the boot FS in `cmdline.txt` the root filesystem is indicated by

```text
console=tty0 console=ttyS1,115200 root=LABEL=RASPIROOT rw fsck.repair=yes net.ifnames=0  rootwait
```

Aside: This has resulted in the wrong root FS benign mounted when multiple media are installed, each with a partitioned named `RASPIROOT`. Note: This is a Debian install. The corresponding line on a RpiOS is

```text
console=serial0,115200 console=tty1 root=PARTUUID=f541a9d1-02 rootfstype=ext4 fsck.repair=yes rootwait quiet splash plymouth.ignore-serial-consoles cfg80211.ieee80211_regdom=US
```

Using `PARTUUID` seems like a better identity for the root partition. Is changing it sufficient or does the initramfs need to be rebuilt. Change `cmdline.txt` is all that is needed on Debian Trixie.

## 2024-06-30 trial

### Platform

* Pi 4B/2GB
* Utility host - RpiOS with desktop on an SD card.
* Target - 520GB Inland SATA SSD.
* Candidate OS - RpiOS, Debian (possibly Ubuntu, Alma, Fedora)

### Plan

1. Install RpiOS (desktop w/out extra apps) on the target SATA SSD (using Imager.)
1. Boot the new install and update/upgrade.
1. Adjust partition size to 40GB and create additional 40GB partitions.

### execution

1. Install `2024-03-15-raspios-bookworm-arm64.img.xz` to target SSD.
1. Install to SSD that previously had ZFS partition failed at resize. Attempt to fix the partition table with `gparted` not successful. Kept winding up at an `(initramfs)` (IIRC) prompt.
1. Secure erased the SSD using a SATA bay on `olive` and repeated install.
1. Following successful boot to new install `mb1` updated and upgraded S/W and rebooted. All good.
1. Booted SD card and resized the root partition and rebooted `mb1`. All good. Boot SD card to perform further operations.
1. Prepare a partition for the next "install" and make a backup copy of the boot partition.

```text
cd /media/hbarta/rootfs/root/boot-backups/mb1
tar tar cf boot.tar /media/hbarta/bootfs
gzip boot.tar
```

```text
root@pi64util:/media/hbarta/rootfs/root/boot-backups/mb1# tar cf boot.tar /media/hbarta/bootfs
tar: Removing leading `/' from member names
root@pi64util:/media/hbarta/rootfs/root/boot-backups/mb1# ls -l
total 76280
-rw-r--r-- 1 root root 78110720 Jun 30 14:37 boot.tar
root@pi64util:/media/hbarta/rootfs/root/boot-backups/mb1# gzip boot.tar 
root@pi64util:/media/hbarta/rootfs/root/boot-backups/mb1# ls -l
total 65080
-rw-r--r-- 1 root root 66637504 Jun 30 14:37 boot.tar.gz
root@pi64util:/media/hbarta/rootfs/root/boot-backups/mb1# 
```

Didn't gain a lot by compressing the boot partition files. Eventually this will need to be done in the target environment.

Next, install Debian to an SD card to use for the 2nd OS. Initial install performed on Debian Laptop using Ansible. Target is Samsung Pro Endurance (black/white) 128GB SD card. Space not needed but speed is helpful.

```text
ansible-playbook provision-Debian-lite.yml -b -K --extra-vars "ssd_dev=/dev/mmcblk0 \
    os_image=/home/hbarta/Downloads/Pi/Debian/20231109_raspi_4_bookworm.img.xz \
    new_host_name=mb2 part_prefix=p\
    eth_hw_mac=dc:a6:32:bf:65:b5 eth_spoof_mac=dc:a6:32:bf:65:b7 \
    wifi_hw_mac=dc:a6:32:bf:65:b6 wifi_spoof_mac=dc:a6:32:bf:65:b8"
```

Now w/out writing image.

```text
ansible-playbook provision-Debian-lite.yml -b -K --extra-vars "ssd_dev=/dev/mmcblk0 \
    new_host_name=mb2 part_prefix=p\
    eth_hw_mac=dc:a6:32:bf:65:b5 eth_spoof_mac=dc:a6:32:bf:65:b7 \
    wifi_hw_mac=dc:a6:32:bf:65:b6 wifi_spoof_mac=dc:a6:32:bf:65:b8"
```
