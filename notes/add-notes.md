# add notes

Manual addition of an additional partition.

## 2024-07-17 find new partition start

Testing initially on a Pi 5 with the host OS (RpiOS) running from NVME SSD and testing on a 64GB SD card.

Start with a new RpiOS install followed by `pmb-init` and first boot. Installing `2024-03-15-raspios-bookworm-arm64.img.xz` and then running

```text
sudo device=/dev/mmcblk0 size=20GiB  ./pmb-init
```

Boot OK and shutdown to reboot (NVME) host. Resulting partition table looks like:

```text
root@wengi:/# sfdisk --list /dev/mmcblk0
Disk /dev/mmcblk0: 59.69 GiB, 64088965120 bytes, 125173760 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x6d3878a4

Device         Boot    Start       End  Sectors  Size Id Type
/dev/mmcblk0p1          8192   1056767  1048576  512M  c W95 FAT32 (LBA)
/dev/mmcblk0p2       1056768  42999807 41943040   20G 83 Linux
/dev/mmcblk0p3      42999808 125173759 82173952 39.2G  5 Extended
root@wengi:/# 
```

In this case, the starting point for the next partition is the same as the extended partition and the partition number will be 5. Now to create a partition starting there and using 10GiB of storage.

```text
sfdisk -s /dev/mmcblk0 -N 5  42999808s 10GiB
sfdisk -s -N 5 /dev/mmcblk0 42999808s 10GiB
echo 'size=10G, start=42999808s, type=L ' | sfdisk -N 5 /dev/mmcblk0
echo -e 'type=L' | sfdisk -N 5 /dev/mmcblk0 
sfdisk -N 5 --delete /dev/mmcblk0  # removed all partitioning!
echo -e 'type=L, size=10GiB' | sfdisk -N 5 /dev/mmcblk0 
```

Reload RpiOS and run 

```text
sudo device=/dev/mmcblk0 size=20GiB  ./pmb-init
```

Skip the boot process and go straight to adding.

```text
echo -e 'type=L, size=10GiB' | sfdisk -N 5 /dev/mmcblk0 
```

Works and results in 

```text
root@wengi:/# echo -e 'type=L, size=10GiB' | sfdisk -N 5 /dev/mmcblk0 
warning: /dev/mmcblk0: partition 5 is not defined yet
Checking that no-one is using this disk right now ... OK

Disk /dev/mmcblk0: 59.69 GiB, 64088965120 bytes, 125173760 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x617a2abd

Old situation:

Device         Boot    Start       End   Sectors  Size Id Type
/dev/mmcblk0p1          8192   1056767   1048576  512M  c W95 FAT32 (LBA)
/dev/mmcblk0p2       1056768  22028287  20971520   10G 83 Linux
/dev/mmcblk0p3      22028288 125173759 103145472 49.2G  5 Extended

/dev/mmcblk0p5: Created a new partition 5 of type 'Linux' and of size 10 GiB.

New situation:
Disklabel type: dos
Disk identifier: 0x617a2abd

Device         Boot    Start       End   Sectors  Size Id Type
/dev/mmcblk0p1          8192   1056767   1048576  512M  c W95 FAT32 (LBA)
/dev/mmcblk0p2       1056768  22028287  20971520   10G 83 Linux
/dev/mmcblk0p3      22028288 125173759 103145472 49.2G  5 Extended
/dev/mmcblk0p5      22030336  43001855  20971520   10G 83 Linux

The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.
root@wengi:/# 
```

Format the new partition.

```text
mkfs.ext4 /dev/mmcblk0p5
```

Expand a Ubuntu image to `/tmp` so we can mount it.

```text
unxz -k --stdout ubuntu-24.04-preinstalled-server-arm64+raspi.img.xz >/tmp/$(basename -s .xz ubuntu-24.04-preinstalled-server-arm64+raspi.img.xz)
```

Create loop device (following pattern at <https://openzfs.github.io/openzfs-docs/Getting%20Started/Ubuntu/Ubuntu%2020.04%20Root%20on%20ZFS%20for%20Raspberry%20Pi.html#step-1-disk-formatting> step #9)

```text
IMG=$(losetup -fP --show /tmp/ubuntu-24.04-preinstalled-server-arm64+raspi.img)
echo $IMG
ls -l ${IMG}*
mkdir /mnt/$(basename ${IMG}p1)
mkdir /mnt/$(basename ${IMG}p2)
mount ${IMG}p1 /mnt/$(basename ${IMG}p1)
mount ${IMG}p2 /mnt/$(basename ${IMG}p2)
```

```text
root@wengi:/# ls -l ${IMG}*
brw-rw---- 1 root disk   7, 0 Jul 17 22:05 /dev/loop0
brw-rw---- 1 root disk 259, 4 Jul 17 22:05 /dev/loop0p1
brw-rw---- 1 root disk 259, 5 Jul 17 22:05 /dev/loop0p2
root@wengi:/# 
root@wengi:/# df -h /mnt/loop*
Filesystem      Size  Used Avail Use% Mounted on
/dev/loop0p1    505M   85M  420M  17% /mnt/loop0p1
/dev/loop0p2    2.8G  1.9G  782M  71% /mnt/loop0p2
root@wengi:/# 
```

Mount the destination partitio0n

```text
mkdir -p /mnt/root
mount /dev/mmcblk0p5 /mnt/root
```

```text
root@wengi:/# mkdir -p /mnt/root
root@wengi:/# mount /dev/mmcblk0p5 /mnt/root
root@wengi:/# df -h /mnt/root
Filesystem      Size  Used Avail Use% Mounted on
/dev/mmcblk0p5  9.8G   24K  9.3G   1% /mnt/root
root@wengi:/# 
```

Copy source root part to destination

```text
rsync -a /mnt/loop0p2/ /mnt/boot
```

Files in `/mnt/root` look right. Now "backup" contents of the boot filesystem to the root filesystem in `root`'s files.

```text
mkdir -p /mnt/root/root-backup/
tar cf /mnt/root/root-backup/mnt-loop0p1-backup.tar /mnt/loop0p1
```

Contents of `/mnt/root/root-backup/mnt-loop0p1-backup.tar` look right. Unmount all and destroy loop device.

```text
umount /mnt/root /mnt/loop0p2 /mnt/loop0p1
losetup -d $IMG
```

## 2024-07-18 try a swap

Mounted root on the SD card from the NVME host to modify the hostname and remap the MAC. Booted into the SD RpiOS install and assigned a static IP. The alternate root was automatically mounted (not particularly desirable ...) Need to back up the current boot partition and restore the other boot partition (and check the `cmdline.txt` and incoming `/etc/fstab`.) Does everything look good in the current (original) install? I think so:

```text
root@mbsd1:~# cat /etc/fstab
proc            /proc           proc    defaults          0       0
PARTUUID=ba34081c-01  /boot/firmware  vfat    defaults          0       2
PARTUUID=ba34081c-02  /               ext4    defaults,noatime  0       1
# a swapfile is not a swap partition, no line here
#   use  dphys-swapfile swap[on|off]  for that
root@mbsd1:~# cat /boot/firmware/cmdline.txt 
console=serial0,115200 console=tty1 root=PARTUUID=ba34081c-02 rootfstype=ext4 fsck.repair=yes rootwait quiet splash plymouth.ignore-serial-consoles cfg80211.ieee80211_regdom=USroot@mbsd1:~# 
root@mbsd1:~# 
root@mbsd1:~# 
root@mbsd1:~# blkid
/dev/nvme0n1p3: LABEL="P22163770E7C5" UUID="10853186896656006072" UUID_SUB="5044967767428183522" BLOCK_SIZE="4096" TYPE="zfs_member" PARTUUID="f541a9d1-03"
/dev/nvme0n1p1: LABEL_FATBOOT="bootfs" LABEL="bootfs" UUID="3A1A-EC0C" BLOCK_SIZE="512" TYPE="vfat" PARTUUID="f541a9d1-01"
/dev/nvme0n1p2: LABEL="rootfs" UUID="063c26d0-665d-497a-ad0f-965d05b228be" BLOCK_SIZE="4096" TYPE="ext4" PARTUUID="f541a9d1-02"
/dev/mmcblk0p5: UUID="1240a8bb-bc9b-449a-91fe-e44d52b90f60" BLOCK_SIZE="4096" TYPE="ext4" PARTUUID="ba34081c-05"
/dev/mmcblk0p1: LABEL_FATBOOT="bootfs" LABEL="bootfs" UUID="50C8-AEAE" BLOCK_SIZE="512" TYPE="vfat" PARTUUID="ba34081c-01"
/dev/mmcblk0p2: LABEL="rootfs" UUID="fc7a1f9e-4967-4f41-a1f5-1b5927e6c5f9" BLOCK_SIZE="4096" TYPE="ext4" PARTUUID="ba34081c-02"
root@mbsd1:~# blkid|grep ba34081c-02
/dev/mmcblk0p2: LABEL="rootfs" UUID="fc7a1f9e-4967-4f41-a1f5-1b5927e6c5f9" BLOCK_SIZE="4096" TYPE="ext4" PARTUUID="ba34081c-02"
root@mbsd1:~# 
```

And backup with a file name distinguishable from the one used for an added install.

```text
mkdir -p /root/root-backup/
tar cf /root/root-backup/boot-firmware-backup.tar /boot/firmware
tar tf /root/root-backup/boot-firmware-backup.tar
```

Result looks right. Now to replace contents of `/boot/firmware` with the added install. First the mount.

```text
alt=/media/hbarta/1240a8bb-bc9b-449a-91fe-e44d52b90f60
tar tf ${alt}/root/root-backup/mnt-loop0p1-backup.tar
```

Backup found in unexpected location

```text
root@mbsd1:~# find ${alt} -name mnt-loop0p1-backup.tar
/media/hbarta/1240a8bb-bc9b-449a-91fe-e44d52b90f60/root-backup/mnt-loop0p1-backup.tar
root@mbsd1:~# 
```

Also it will be necessary to fixup `cmdline.txt` and `/etc/fstab`. `cmdline.txt` cannot be done ahead of time because that partition is directly backed up form the install image. It would be possible to fixup the `/etc/fstab` after it is copied to the target media.

```text
alt=/media/hbarta/1240a8bb-bc9b-449a-91fe-e44d52b90f60
tar tf ${alt}/root-backup/mnt-loop0p1-backup.tar
cd /boot/firmware
rm -rf *
tar -xvf ${alt}/root-backup/mnt-loop0p1-backup.tar --strip-components=2
blkid
```

```text
root@mbsd1:/boot/firmware# blkid
/dev/nvme0n1p3: LABEL="P22163770E7C5" UUID="10853186896656006072" UUID_SUB="5044967767428183522" BLOCK_SIZE="4096" TYPE="zfs_member" PARTUUID="f541a9d1-03"
/dev/nvme0n1p1: LABEL_FATBOOT="bootfs" LABEL="bootfs" UUID="3A1A-EC0C" BLOCK_SIZE="512" TYPE="vfat" PARTUUID="f541a9d1-01"
/dev/nvme0n1p2: LABEL="rootfs" UUID="063c26d0-665d-497a-ad0f-965d05b228be" BLOCK_SIZE="4096" TYPE="ext4" PARTUUID="f541a9d1-02"
/dev/mmcblk0p5: UUID="1240a8bb-bc9b-449a-91fe-e44d52b90f60" BLOCK_SIZE="4096" TYPE="ext4" PARTUUID="ba34081c-05"
/dev/mmcblk0p1: LABEL_FATBOOT="bootfs" LABEL="bootfs" UUID="50C8-AEAE" BLOCK_SIZE="512" TYPE="vfat" PARTUUID="ba34081c-01"
/dev/mmcblk0p2: LABEL="rootfs" UUID="fc7a1f9e-4967-4f41-a1f5-1b5927e6c5f9" BLOCK_SIZE="4096" TYPE="ext4" PARTUUID="ba34081c-02"
root@mbsd1:/boot/firmware# 
```

Fix cmdline.txt and try to boot Ubuntu! Not successful, `/` is mounted ro. There was an error from 

```text
systemd-remount-fs[375]: mount: /: can't find LABEL=`writable`
``` 

Need to recheck the filesystems in the install image and duplicate this for the new partition. Repeating the steps found at "Expand a Ubuntu image" above

```text
cd Downloads/Pi/Ubuntu/
unxz -k --stdout ubuntu-24.04-preinstalled-server-arm64+raspi.img.xz >/tmp/$(basename -s .xz ubuntu-24.04-preinstalled-server-arm64+raspi.img.xz)
sudo -s
cd
IMG=$(losetup -fP --show /tmp/ubuntu-24.04-preinstalled-server-arm64+raspi.img)
blkid |grep loop
# insert SD card
blkid |grep mmc
e2label /dev/mmcblk0p5 writable
```

```text
root@wengi:~# blkid |grep loop
/dev/loop0p1: LABEL_FATBOOT="system-boot" LABEL="system-boot" UUID="F526-0340" BLOCK_SIZE="512" TYPE="vfat" PARTUUID="0529037a-01"
/dev/loop0p2: LABEL="writable" UUID="1305c13b-200a-49e8-8083-80cd01552617" BLOCK_SIZE="4096" TYPE="ext4" PARTUUID="0529037a-02"
root@wengi:~# 

root@wengi:~# blkid |grep mmc
/dev/mmcblk0p1: LABEL_FATBOOT="bootfs" LABEL="bootfs" UUID="50C8-AEAE" BLOCK_SIZE="512" TYPE="vfat" PARTUUID="ba34081c-01"
/dev/mmcblk0p2: LABEL="rootfs" UUID="fc7a1f9e-4967-4f41-a1f5-1b5927e6c5f9" BLOCK_SIZE="4096" TYPE="ext4" PARTUUID="ba34081c-02"
/dev/mmcblk0p5: LABEL="writable" UUID="1240a8bb-bc9b-449a-91fe-e44d52b90f60" BLOCK_SIZE="4096" TYPE="ext4" PARTUUID="ba34081c-05"
root@wengi:~# losetup -d $IMG
root@wengi:~# 

```

OK, try again. Notice that /boot/firmware was not mounted. (Captured picture) but other stuff seems to be proceeding. It seems likely desirable to tweak the Systemd jobs that mount boot and root filesystems to do so by UUID. Boot process halted again on failure to mount `/boot/firmware`. Mounted manually and `<ctrl>D` allowed boot to complete. Woo! Need to lookup default login for Ubuntu server. S/B `ubuntu`:`ubuntu` but that's not working. :-/. May need to clear the password on the SD install and try again. Pushbutton works - Nice! Foumd `!` in the password field for `ubuntu` in `/etc/shadow` and removed it. Rebooting. Still hanging on mount for `/boot/firmware`. Login as `ubuntu` with no password prompt.

Tracking sown the Systemd units that borked on mounting boot and root, I see they are both in `/run` (AKA `tmpfs`) and must be created during the boot process. I tweaked `/etc/fstab` to use `PARTUUID` to ID partitions and the subsequent reboot was uneventful. I wonder if I can use the label to ID the partition.

```text
e2label /dev/mmcblk0p5 ubuntu
```

Yes! That works. Next step, remap the MAC address.

List of things to do fpr Ubuntu server

* `PARTUUID` for `/etc/fstab`
* remap MAC - really troublesome. Try removing cloud-init per <https://openzfs.github.io/openzfs-docs/Getting%20Started/Ubuntu/Ubuntu%2022.04%20Root%20on%20ZFS%20for%20Raspberry%20Pi.html#step-6-full-software-installation>
* hostname
* timezone
* add user
* consider replacing Netplan with systemd-networkd.
* remove passwordless `sudo`

## 2024-07-18 switch back to RpiOS

```text
mkdir -p /root/root-backup/
tar cf /root/root-backup/boot-firmware-backup.tar /boot/firmware
tar tf /root/root-backup/boot-firmware-backup.tar
```

And restore the RpiOS stuff

```text
mkdir -p /mnt/mmcblk0p2
mount /dev/mmcblk0p2 /mnt/mmcblk0p2 
tar tf /mnt/mmcblk0p2/root/root-backup/boot-firmware-backup.tar
cd /
rm -r /boot/firmware/*
tar xf /mnt/mmcblk0p2/root/root-backup/boot-firmware-backup.tar
shutdown -r now
```

And "Bob's your uncle" RpiOS is up and running. Next, thrash the switching process a bit. And ...

Issue: Ubuntu expanded to use the rest of the space in the Extended partition. Two potential solution:

1. Create a "blocker" partition following the new Logical partition.
1. Create the new Logical partition at the end of the Extended partition. (seems better, just need to calculate the apopropriate starting sector or create the partition and then move it to the end of the Extended partition.)

Full update/upgrade for RpiOS. Seems like all is good.

And restore Ubuntu

```text
tar tf /media/hbarta/ubuntu/root/root-backup/boot-firmware-backup.tar
cd /
rm -r /boot/firmware/*
tar xf /media/hbarta/ubuntu/root/root-backup/boot-firmware-backup.tar
shutdown -r now
```

Full update/upgrade and all seems well. Back to `RpiOS`. And that works too.

## 2024-07-18 allocating space for additional OSs

Need a strategy for carving up the additional space (in the Extended partition) for additional OSs. As an aside, it also appears that each OS may require special handling to work. Examples include:

* Debian requires a logical partition at the end of the extended partition.
* Ubuntu requires modification of Netplan configuration for MAC to support MAC spoofing.
* Fedora (not even tried yet) starts with three partitions, FAT, EXT4 and btrfs.

Possible strategy:

1. (During init), create a logical partition that uses the entire Extended partition, label it `filler` presuming an empty partition vsn be labeled. Otherwise determine how to ID a logical partition with no format.
1. Reduce the size of the `filler` partition by the space desired for the addition.
1. Create a new logical partition in the available unused space, format it EXT4 and use for the addition.

### Playing with partitioning

Using a SATA SSD in a hot swap bay in an X86_64 desktop for convenience and speed. Drive comes in as `/dev/sdb` and is half of the previous system drive in a server. Just secure erase and move on.

```text
hdparm --user-master u --security-set-pass Eins /dev/sdb
hdparm --security-erase Eins /dev/sdb
```

(Installed RpiOS using `rpi-imager` in `wengi` and back to `olive` w/out booting.)

```text
root@olive:~# sfdisk --list /dev/sdb
Disk /dev/sdb: 223.57 GiB, 240057409536 bytes, 468862128 sectors
Disk model: ST240HM000-1G515
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes
Disklabel type: dos
Disk identifier: 0x617a2abd

Device     Boot    Start       End   Sectors   Size Id Type
/dev/sdb1           8192   1056767   1048576   512M  c W95 FAT32 (LBA)
/dev/sdb2        1056768  42999807  41943040    20G 83 Linux
/dev/sdb3       42999808 468862127 425862320 203.1G  5 Extended
root@olive:~# 
```

```text
echo -e 'type=L' | sfdisk -N 5 /dev/sdb
```

Previous command creates a logical partition that uses all of the Extended partition.

```text
Device     Boot    Start       End   Sectors   Size Id Type
/dev/sdb1           8192   1056767   1048576   512M  c W95 FAT32 (LBA)
/dev/sdb2        1056768  42999807  41943040    20G 83 Linux
/dev/sdb3       42999808 468862127 425862320 203.1G  5 Extended
/dev/sdb5       43001856 468862127 425860272 203.1G 83 Linux
```

```text
root@olive:~# blkid|grep sdb
/dev/sdb1: LABEL_FATBOOT="bootfs" LABEL="bootfs" UUID="50C8-AEAE" BLOCK_SIZE="512" TYPE="vfat" PARTUUID="617a2abd-01"
/dev/sdb2: LABEL="rootfs" UUID="fc7a1f9e-4967-4f41-a1f5-1b5927e6c5f9" BLOCK_SIZE="4096" TYPE="ext4" PARTUUID="617a2abd-02"
/dev/sdb5: PARTUUID="617a2abd-05"
root@olive:~# 
```

`blkid` does not list the Extended partition and lists no `TYPE=` field for the logical partition (along with no `LABEL`, `UUID`, `BLOCK_SIZE` and `PARTUUID`). Now to to try to resize

```text
echo -e 'size=325860272' | sfdisk -N 5 /dev/sdb
```

This creates empty space at the end of the partition, but involves some gnarly arithmetic to get the desired space. A better strategy is to delete the `filler`, create the desired space and recreate the filler with remaining space. That provides the advantage that the required space can be provided in any format accepted by `sfdisk`.

```text
echo -e 'type=L' | sfdisk -N 5 /dev/sdb
sfdisk --delete /dev/sdb 5
echo -e 'type=L, size=10GiB' | sfdisk -N 5 /dev/sdb 
##  BAD BAD BAD echo -e 'type=L' | sfdisk /dev/sdb
echo -e 'type=L' | sfdisk -N 6 /dev/sdb
```

That "BAD" one just wiped the partition table and replaced it with a single partition. Need to specify the partition number. Time to return attention to `pmb-add`.

## 2024-08-09 I'm back

After setting this aside for a few days, I'm returning to it. My recollection is that the `pmb-add` script was working until the last commit. (Note to self - DO NOT COMMIT UNTIL I HAVE TESTED!) Reviewing the last commit at <https://github.com/HankB/pmb/commit/16b15d77b65553d56897268ad1f33e051b7b2450> (?). Looks like a `$` was lost on line 60. Still testing with `/dev/sdb`:

```text
device=/dev/sdb size=10G image=/home/hbarta/Downloads/Pi/Raspbian/image_2023-09-28-raspios-bookworm-arm64-lite.img.xz ./pmb-add
```

This test proceeded with the only exception being a prompt from `mke2fs` about an existing filesystem

```text
mke2fs 1.47.0 (5-Feb-2023)
/dev/sdb12 contains a ext4 file system
        created on Fri Jul 19 17:04:14 2024
Proceed anyway? (y,N) y
```

This seems to be a result of the error caused during a previous run. Answered `y` and continued. A subsequent completed w/out any warnings. Now need to re-image the first OS and repeat this test.

```text
smartctl -a /dev/sdb
blkdiscard -f /dev/sdb
xzcat /home/hbarta/Downloads/Pi/Debian/20231109_raspi_4_bookworm.img.xz >/dev/sdb
size=10G device=/dev/sdb ./pmb-init
size=10G device=/dev/sdb ./pmb-init
```

Appears to work. (Not sure why I ran `pmb-init` twice ...)

Next step is to actually swap bewtween images to confirm. First test will be RpiOS install first, Debian Bookworm next. (Testing on Pi 5 `wengi`)

1. `rpi-imager` Pi 4, Lite, no WiFi.
1. `device=/dev/sda size=10G ./pmb-init`
1. SSD to Pi 4B, boot - OK
1. SSD to Pi 5
1. `image=/home/hbarta/Downloads/Pi/Debian/20231109_raspi_4_bookworm.img.xz size=12GiB device=/dev/sda ./pmb-add`
1. SSD to Pi 4B and boot - boot OK.
1. SSD to Pi 5

Starting [notes on swap](./swap-notes.md#2024-08-09-manual-swap) for swap. and realize that `pmb-add` is incomplete.

## 2024-08-11 more work for pmb-add

At present the script ends with uncompressing the image to install to `/tmp`. Additional work to do is:

1. Loop mount both partitions in the image. (done)
1. Mount the newly created partition and format to EXT4. (done)
1. Uncompress the image (to be installed) and loop mount both partitions. (done)
1. Copy contents of root partition to the new partition (rsync.) (done)
1. Copy the contents of the boot partition to a directory in the new partition (tar.) (done)
1. source generic helpers and spoof MAC addresses (done)
1. Call any specified helper (done)
1. Add new OS to tracking table. (done)
1. Unmount all partitions and remove mount points. (done)
1. Remove uncompressed image. (done)

## 2024-08-21 testing

The tar which should only include the firmware stuff seems to include everything. Fixed.

Tested `pmb-init` (RpiOS) and `pmb-add` (Debian12) and files for Debian look good. RpiOS upgrading 19 packages and will [manually swap to Debian12](./swap-notes.md#2024-08-21-manual-swap).

`pmb-add` needs to fix `/etc/fstab` at least for Debian, likely for all. But for now will do this in a distro specific helper.

## 2024-08-22 testing, again

latest changes, prepping for `pmb-swap`. RpiOS primary, Debian next.

## 2024-09-23 testing

1. `rpi-imager` to install RpiOS to Kingston SSD `/dev/sda`.
1. `pmb-init` to configure RpiOS.
1. `pmb-add` to add Debian. No joy, complains about wrong partition table.
1. Reboot, This mounts all three partitions on `/dev/sda`.
1. `pmb-add` to add Debian. No joy, complains about mounted partitions.

```text
root@mars:/home/hbarta/Programming/pmb# ./pmb-add vars_Debian_bookworm 
sourcing vars_Debian_bookworm
============================================= find device (part5)
============================================= part5: /dev/sda5
5
6
7
next available partition is 7 and new partition is 6
============================================= partitions to use: 7 and 6
sfdisk: /dev/sda: partition 6: failed to delete
root@mars:/home/hbarta/Programming/pmb# 
```

