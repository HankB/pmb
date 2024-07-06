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
tar cf boot.tar /media/hbarta/bootfs
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

```text
ansible-playbook first-boot-Debian.yml -i inventory -l mb2 -u root
```

This process hung, perhaps due to the excessive number of packages to upgrade. After a couple hours, killed it and rebooted. It was necessary to `dpkg -<whatever finishes things>` to get everything done. The next playbook ran w/out any problems.

```text
ansible-playbook second-boot-bookworm-lite-Debian.yml \
    -i inventory -l mb2 -u root
```

Shutdown and boot from SSD to prepare to copy `mb2` to the SSD and boot that.

Disk situation following boot and SD card insert:

```text
hbarta@mb1:~ $ df
Filesystem     1K-blocks    Used Available Use% Mounted on
udev              671944       0    671944   0% /dev
tmpfs             189024    1520    187504   1% /run
/dev/sda2       40245400 4942596  33238424  13% /
tmpfs             945116     352    944764   1% /dev/shm
tmpfs               5120      16      5104   1% /run/lock
/dev/sda1         522230   76392    445838  15% /boot/firmware
tmpfs             189020      40    188980   1% /run/user/1000
/dev/sda3       40005128      24  37940720   1% /media/hbarta/debian
/dev/mmcblk0p1    519904  145784    374120  29% /media/hbarta/RASPIFIRM
/dev/mmcblk0p2  30418168 1703004  27152156   6% /media/hbarta/RASPIROOT
hbarta@mb1:~ $ 
```

Not happy that `/media/hbarta/debian` mounted automatically. Perhaps can deal with that later. Now for rsync from `/media/hbarta/RASPIROOT` to `/media/hbarta/debian`, From <https://www.baeldung.com/linux/rsync-clone-file-system-hierarchy>

```text
rsync -axHAWXS --numeric-ids --info=progress2 /mnt/sourcePart/ /mnt/destPart
```

From <https://superuser.com/questions/307541/copy-entire-file-system-hierarchy-from-one-drive-to-another>

```text
rsync -avxHAX --progress / /new-disk/
```

First try

```text
rsync -axHAWXS /media/hbarta/RASPIROOT/ /media/hbarta/debian
```

Copy proceeded w/out issue and seems to have achieved the desired result. Boot pool next, `/media/hbarta/RASPIFIRM` to `/boot/firmware` after removing files from `/boot/firmware`

```text
rm -rf /boot/firmware/*
rsync -axHAWXS /media/hbarta/RASPIFIRM/ /boot/firmware
```

Then fix the `cmdline.txt` to point to the block ID of `/dev/sda3`.

```text
root@mb1:~# blkid /dev/sda3
/dev/sda3: LABEL="debian" UUID="2c53b108-22e2-4030-918c-bbc091c21a8a" BLOCK_SIZE="4096" TYPE="ext4" PARTUUID="6a002ed5-03"
root@mb1:~# vi /boot/firmware/cmdline.txt 
root@mb1:~# cat /boot/firmware/cmdline.txt
console=tty0 console=ttyS1,115200 root=PARTUUID=6a002ed5-03 rw fsck.repair=yes net.ifnames=0  rootwait 
root@mb1:~# 
```

### Result

Target does not boot. Did I overlook necessary changes to `/etc/fstab` in `mb2`? Worse, utility install on the SD card no longer boots. :-/ Will fiddle with both to try to determine what I did wrong.

## 2024-07-01 pi64util rescue

Boot looping yesterday. Mounted today in the SD card slot in a CM4 IO board (trixi)

* `cmdline.txt` looks correct. `root=PARTUUID=e10f1597-02` 

```text
/dev/mmcblk0p2: LABEL="rootfs" UUID="fc7a1f9e-4967-4f41-a1f5-1b5927e6c5f9" BLOCK_SIZE="4096" TYPE="ext4" PARTUUID="e10f1597-02"
```

Looks unmolested. Mounting the root partition to check that. 

```text
root@trixi:~# cat /mnt/root/etc/hostname
pi64util
root@trixi:~#
```

Looks right. It does not appear that either partition was overwritten. Something more subtle? No. Today it boots w/out difficulty on a CM4 and 4B (with no other drives attached.)

## 2024-07-01 multiboot rescue

Mounted SSD (running on 4B, `pi64util`) and found `mb2` using partition labels in `/etc/fstab`. Switched to PARTUUID. That works and now `mb2` boots. Next effort is to switch back to `mb1` and will do from `mb2`.

## 2024-07-01 mb2 to mb1 from mb2

Things needed to switch.

1. Backup `mb2`'s boot partition.
1. Remove files from shared boot partition. `/boot/firmware/`.
1. Restore `mb1`'s boot partition.
1. Check `/boot/firmware/cmdline.txt` and `/etc/fstab` to confirm they are correct.

But first install security updates and make sure `mb2` reboots. It does. Mounting the `mb1` root partition produces a curious message (but mounts the partition.):

```text
oot@mb2:/home/hbarta# mkdir /mnt/mb1_boot
root@mb2:/home/hbarta# blkid
/dev/sda2: LABEL="rootfs" UUID="fc7a1f9e-4967-4f41-a1f5-1b5927e6c5f9" BLOCK_SIZE="4096" TYPE="ext4" PARTUUID="6a002ed5-02"
/dev/sda3: LABEL="debian" UUID="2c53b108-22e2-4030-918c-bbc091c21a8a" BLOCK_SIZE="4096" TYPE="ext4" PARTUUID="6a002ed5-03"
/dev/sda1: LABEL_FATBOOT="bootfs" LABEL="bootfs" UUID="50C8-AEAE" BLOCK_SIZE="512" TYPE="vfat" PARTUUID="6a002ed5-01"
root@mb2:/home/hbarta# mv /mnt/mb1_boot /mnt/mb1
root@mb2:/home/hbarta# mount /dev/sda2 /mnt/mb1
mount: (hint) your fstab has been modified, but systemd still uses
       the old version; use 'systemctl daemon-reload' to reload.
root@mb2:/home/hbarta# 
```

```text
cd
mkdir -p boot-backups/mb2
tar cf boot-backups/mb2/boot.tar /boot/firmware
tar tf boot-backups/mb2/boot.tar
ls -l /boot/firmware/
rm -rf /boot/firmware/*
tar --directory=/ /mnt/mb1/root/boot-backups/mb1/boot.tar.gz
```

This tar archive is created with the path `media/hbarta/bootfs/`. Need to

```text
cd /boot/firmware
tar -xvf /mnt/mb1/root/boot-backups/mb1/boot.tar.gz --strip-components=3
```

And that worked. When the boot partition is archived from `mb2`/`mb1` a different command will be required. Everything looks right.

```text
root@mb2:/boot/firmware# cat cmdline.txt
console=serial0,115200 console=tty1 root=PARTUUID=6a002ed5-02 rootfstype=ext4 fsck.repair=yes rootwait quiet splash plymouth.ignore-serial-consolesroot@mb2:/boot/firmware# blkid|grep 6a002ed5-02
/dev/sda2: LABEL="rootfs" UUID="fc7a1f9e-4967-4f41-a1f5-1b5927e6c5f9" BLOCK_SIZE="4096" TYPE="ext4" PARTUUID="6a002ed5-02"
root@mb2:/boot/firmware# cat /mnt/mb1/etc/fstab 
proc            /proc           proc    defaults          0       0
PARTUUID=6a002ed5-01  /boot/firmware  vfat    defaults          0       2
PARTUUID=6a002ed5-02  /               ext4    defaults,noatime  0       1
# a swapfile is not a swap partition, no line here
#   use  dphys-swapfile swap[on|off]  for that
root@mb2:/boot/firmware# 
```

Reboot and Great Success!

## add an OS using an extended partition

Add Debian Trixie installed on another SSD (and with ZFS, but ignoring that for now.)

1. Create the partition using `gparted`. It seems to come up w/ no filesystem. Ha! That's because an "extended partition" is a container and I needed to create logical partitions inside in which to create filesystems.

```text
root@mb1:~# fdisk /dev/sda

Welcome to fdisk (util-linux 2.38.1).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

This disk is currently in use - repartitioning is probably a bad idea.
It's recommended to umount all file systems, and swapoff all swap
partitions on this disk.


Command (m for help): p

Disk /dev/sda: 476.94 GiB, 512110190592 bytes, 1000215216 sectors
Disk model:                 
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 33553920 bytes
Disklabel type: dos
Disk identifier: 0x6a002ed5

Device     Boot     Start        End   Sectors   Size Id Type
/dev/sda1            8192    1056767   1048576   512M  c W95 FAT32 (LBA)
/dev/sda2         1056768   82976767  81920000  39.1G 83 Linux
/dev/sda3        82976768  164896767  81920000  39.1G 83 Linux
/dev/sda4       164896768 1000214527 835317760 398.3G  5 Extended
/dev/sda5       164898816  246818815  81920000  39.1G 83 Linux

Command (m for help): q

root@mb1:~# mkdir /mnt/sda5 
root@mb1:~# mount /dev/sda5 /mnt/sda5
root@mb1:~# rsync -axHAWXS --numeric-ids --info=progress2 /mnt/sdb2 /mnt/sda5
rsync: [sender] link_stat "/mnt/sdb2" failed: No such file or directory (2)
              0 100%    0.00kB/s    0:00:00 (xfr#0, to-chk=0/0)
rsync error: some files/attrs were not transferred (see previous errors) (code 23) at main.c(1338) [sender=3.2.7]
root@mb1:~# rsync -axHAWXS --numeric-ids --info=progress2 /media/hbarta/RASPIROOT/ /mnt/sda5
  3,049,131,170  99%   50.33MB/s    0:00:57 (xfr#93051, to-chk=0/106257)   
root@mb1:~# 
```

Backup the `mb1` boot partition

```text
root@mb1:~# tar cf boot-backups/mb1/boot.tar /boot/firmware
tar: Removing leading `/' from member names
root@mb1:~# 
```

Copy the `trixie` boot partition to a cleared `/boot/firmware` partition.

```text
root@mb1:~# rm -rf /boot/firmware/*
root@mb1:~#  rsync -axHAWXS --numeric-ids  /media/hbarta/RASPIFIRM/ /boot/firmware
rsync: [generator] chown "/boot/firmware/." failed: Operation not permitted (1)
rsync: [generator] chown "/boot/firmware/boot" failed: Operation not permitted (1)
...
rsync: [receiver] chown "/boot/firmware/overlays/.wm8960-soundcard.dtbo.PtIkL3" failed: Operation not permitted (1)
rsync error: some files/attrs were not transferred (see previous errors) (code 23) at main.c(1338) [sender=3.2.7]
root@mb1:~# ls -l /boot/firmware
total 242538
-rwxr-xr-x 1 root root    38318 Jul  5 17:13 bcm2711-rpi-4-b.dtb
-rwxr-xr-x 1 root root    38238 Jul  5 17:13 bcm2711-rpi-400.dtb
-rwxr-xr-x 1 root root    38325 Jul  5 17:13 bcm2711-rpi-cm4-io.dtb
-rwxr-xr-x 1 root root    20713 Jul  5 17:13 bcm2837-rpi-3-a-plus.dtb
-rwxr-xr-x 1 root root    21443 Jul  5 17:13 bcm2837-rpi-3-b-plus.dtb
-rwxr-xr-x 1 root root    20979 Jul  5 17:13 bcm2837-rpi-3-b.dtb
-rwxr-xr-x 1 root root    20118 Jul  5 17:13 bcm2837-rpi-cm3-io3.dtb
-rwxr-xr-x 1 root root    20544 Jul  5 17:13 bcm2837-rpi-zero-2-w.dtb
drwxr-xr-x 3 root root     2048 May  7 10:46 boot
-rwxr-xr-x 1 root root    52476 Jul  5 17:13 bootcode.bin
-rwxr-xr-x 1 root root       99 Jul  5 17:13 cmdline.txt
-rwxr-xr-x 1 root root      652 Jul  5 17:13 config.txt
-rwxr-xr-x 1 root root     7303 Jul  5 17:13 fixup.dat
-rwxr-xr-x 1 root root     5436 Jul  5 17:13 fixup4.dat
-rwxr-xr-x 1 root root     3204 Jul  5 17:13 fixup4cd.dat
-rwxr-xr-x 1 root root     8425 Jul  5 17:13 fixup4db.dat
-rwxr-xr-x 1 root root     8425 Jul  5 17:13 fixup4x.dat
-rwxr-xr-x 1 root root     3204 Jul  5 17:13 fixup_cd.dat
-rwxr-xr-x 1 root root    10266 Jul  5 17:13 fixup_db.dat
-rwxr-xr-x 1 root root    10268 Jul  5 17:13 fixup_x.dat
-rwxr-xr-x 1 root root 52152279 Jul  5 17:13 initrd.img-6.6.15-arm64
-rwxr-xr-x 1 root root 32614032 Jul  5 17:13 initrd.img-6.7.12-arm64
-rwxr-xr-x 1 root root 32705228 Jul  5 17:13 initrd.img-6.8.12-arm64
drwxr-xr-x 2 root root     4096 May  7 10:47 old
drwxr-xr-x 2 root root    24576 Aug 24  2023 overlays
-rwxr-xr-x 1 root root  2981056 Jul  5 17:13 start.elf
-rwxr-xr-x 1 root root  2256800 Jul  5 17:13 start4.elf
-rwxr-xr-x 1 root root   809468 Jul  5 17:13 start4cd.elf
-rwxr-xr-x 1 root root  3754024 Jul  5 17:13 start4db.elf
-rwxr-xr-x 1 root root  3004488 Jul  5 17:13 start4x.elf
-rwxr-xr-x 1 root root   809468 Jul  5 17:13 start_cd.elf
-rwxr-xr-x 1 root root  4825896 Jul  5 17:13 start_db.elf
-rwxr-xr-x 1 root root  3728104 Jul  5 17:13 start_x.elf
-rwxr-xr-x 1 root root     1076 Jul  5 17:13 sysconf.txt
-rwxr-xr-x 1 root root 35497920 Jul  5 17:13 vmlinuz-6.6.15-arm64
-rwxr-xr-x 1 root root 36192192 Jul  5 17:13 vmlinuz-6.7.12-arm64
-rwxr-xr-x 1 root root 36630464 Jul  5 17:13 vmlinuz-6.8.12-arm64
root@mb1:~# 
```

Now to fixup `cmdline.txt` and `fstab`

```text
root@mb1:~# blkid |grep sda5
/dev/sda5: LABEL="trixie" UUID="3f246d49-ce8b-4d21-87a4-1960f825c7e3" BLOCK_SIZE="4096" TYPE="ext4" PARTUUID="6a002ed5-05"
root@mb1:~# 
root@mb1:~# cat /boot/firmware/cmdline.txt
console=tty0 console=ttyS1,115200 root=PARTUUID=6a002ed5-05 rw fsck.repair=yes net.ifnames=0  rootwait 
root@mb1:~# cat /mnt/sda5/etc/fstab
# The root file system has fs_passno=1 as per fstab(5) for automatic fsck.
PARTUUID=6a002ed5-05 / ext4 rw 0 1
# All other file systems have fs_passno=2 as per fstab(5) for automatic fsck.
PARTUUID=6a002ed5-01 /boot/firmware vfat rw 0 2
root@mb1:~# 
```

Forgot to update the MAC address but it came up anyway as `neutron`. And it boots from a logical pattition w/out difficulty.
