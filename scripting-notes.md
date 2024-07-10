# scripting notes

Trying out commands that will be used to script some operations.

## 2024-07-06 pmb-init.sh

Initial conditions: Pi image written to the media (SD card) and before booting. Operations required.

1. Expand EXT4 partition to predetermined size - 10GB. (Expect EXT4 FS to be expanded on first boot.)
1. Create extended partition using the rest of the media.

Choices `sfdisk` and `parted` are both scriptable.

* <https://www.man7.org/linux/man-pages/man8/sfdisk.8.html> No obvious way to expand a partition, probably need to delete and recreate at the new size. May need to capture the starting address and use that when creating.
* <https://www.gnu.org/software/parted/manual/parted.html>

```text
root@rocinante:/home/hbarta/Programming/pmb# parted /dev/mmcblk0 print
Model: SD SH32G (sd/mmc)
Disk /dev/mmcblk0: 31.9GB
Sector size (logical/physical): 512B/512B
Partition Table: msdos
Disk Flags: 

Number  Start   End     Size    Type     File system  Flags
 1      4194kB  273MB   268MB   primary  fat32        lba
 2      273MB   2101MB  1829MB  primary  ext4

root@rocinante:/home/hbarta/Programming/pmb# 

root@rocinante:/home/hbarta/Programming/pmb# parted -l -m /dev/mmcblk0 
BYT;
/dev/nvme0n1:1024GB:nvme:512:512:gpt:HP SSD EX950 1TB:;
2:1049kB:538MB:537MB:fat32::boot, esp;
3:538MB:1612MB:1074MB:zfs::;
4:1612MB:1024GB:1023GB:zfs::;

BYT;
/dev/mmcblk0:31.9GB:sd/mmc:512:512:msdos:SD SH32G:;
1:4194kB:273MB:268MB:fat32::lba;
2:273MB:2101MB:1829MB:ext4::;

BYT;
/dev/zd0:4295MB:unknown:512:4096:loop:Unknown:;
1:0.00B:4295MB:4295MB:linux-swap(v1)::;

root@rocinante:/home/hbarta/Programming/pmb# 
```

```text
root@rocinante:/home/hbarta/Programming/pmb# s
Disk /dev/mmcblk0: 29.72 GiB, 31914983424 bytes, 62333952 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x544c6228

Device         Boot  Start     End Sectors  Size Id Type
/dev/mmcblk0p1        8192  532479  524288  256M  c W95 FAT32 (LBA)
/dev/mmcblk0p2      532480 4104191 3571712  1.7G 83 Linux
root@rocinante:/home/hbarta/Programming/pmb# 
```

Prototype commands to delete partition 2 and recreate it at 10GB.

```text
sfdisk --delete /dev/mmcblk0 2          # not needed
echo  'size=10G, type=L' | sfdisk -N 2 /dev/mmcblk0 # works
echo  ';' | sfdisk -N 3 /dev/mmcblk0 # no
echo "+,+" | sfdisk -N 3 /dev/mmcblk0 # no
```

```text
parted  /dev/mmcblk0 resizepart 2 10G
parted /dev/mmcblk0 mkpart extended 100%
```

## 2024-07-07 more exploration

Status: Following command expands the partition and RpiOS still boots and then expands to fill remaining space. Attempts to use `sfdisk` to use the remaining space (before booting) to create an extended partition not yet successful. `sfdisk` sees enough space before the first partition and creates a tiny extended partition there.

```text
echo  'size=10G, type=L' | sfdisk -N 2 /dev/mmcblk0 # works
```

As it seems necessary to identify the first sector to use for a new partition, the following will do that.

```text
s=$(($(sfdisk --list /dev/mmcblk0 |grep /dev/mmcblk0p2|awk '{print $3}')+1))
echo $s
```

```text
echo  "start=$s type=Ex" | sfdisk -N 3 /dev/mmcblk0 # works! and boots/resizes
```

Now the logical partition for the new install. Following does not work.

```text
echo  'size=10G, type=85' | sfdisk -N 4 /dev/mmcblk0
```

(Switching to another PC, SD card now `/dev/sdb`.)

```text
s=$(($(sfdisk --list /dev/sdb |grep /dev/sdb2|awk '{print $3}')+1))
echo $s
```

```text
root@olive:~# s=$(($(sfdisk --list /dev/sdb |grep /dev/sdb2|awk '{print $3}')+1))
root@olive:~# echo $s
19531251
root@olive:~# 
```

Following works (for first logical partition.)

```text
echo  "type=L size=10G" | sfdisk -N 5 /dev/sdb 
```

NB: Start logical partitions at #5.

## 2024-07-07 partition numbers

`/dev/sdb2` vs. `/dev/mmcblk0p2` - see the problem? Given a device, the script needs to ID the partiti0on numbering policy in effect. Testing for the existence of the device name for the 2nd partition should suffice.

## 2024-07-08 testing initial pmb-init

Works with RpiOS. Hangs with Debian. Perhaps pre-expanding the root partition is an issue. Here's the test situation (Debian image)

```text
root@wengi:/home/hbarta# sfdisk -p /dev/mmcblk0
sfdisk: invalid option -- 'p'
Try 'sfdisk --help' for more information.
root@wengi:/home/hbarta# sfdisk -l /dev/mmcblk0
Disk /dev/mmcblk0: 29.72 GiB, 31914983424 bytes, 62333952 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x388501c0

Device         Boot    Start      End  Sectors  Size Id Type
/dev/mmcblk0p1          8192  1048575  1040384  508M  c W95 FAT32 (LBA)
/dev/mmcblk0p2       1048576  5119999  4071424  1.9G 83 Linux
/dev/mmcblk0p3      16334848 62332927 45998080 21.9G  5 Extended
root@wengi:/home/hbarta# 
```

Still wrestling with Debian, trying the Ansible setup

```text
ansible-playbook provision-Debian-lite.yml -b -K --extra-vars "ssd_dev=/dev/mmcblk0 \
    os_image=/home/hbarta/Downloads/ISO/Debian/20231109_raspi_4_bookworm.img.xz \
    new_host_name=mb1 poolname=mb1_tank part_prefix=p\
    eth_hw_mac=dc:a6:32:bf:65:b5 eth_spoof_mac=dc:a6:32:bf:65:b7 \
    wifi_hw_mac=dc:a6:32:bf:65:b6 wifi_spoof_mac=dc:a6:32:bf:65:b8"
```

```text
ansible-playbook provision-Debian-lite.yml -b -K --extra-vars "ssd_dev=/dev/mmcblk0 \
    new_host_name=mb1 poolname=mb1_tank part_prefix=p\
    eth_hw_mac=dc:a6:32:bf:65:b5 eth_spoof_mac=dc:a6:32:bf:65:b7 \
    wifi_hw_mac=dc:a6:32:bf:65:b6 wifi_spoof_mac=dc:a6:32:bf:65:b8"
```

## 2024-07-10 Debian first boot woes

Unable to find any combo that works here. The Ansible playbook that does partition manipulation and works uses `parted` so will proceed testing that.

```text
xzcat /home/hbarta/Downloads/ISO/Debian/20231109_raspi_4_bookworm.img.xz >/dev/mmcblk0
parted -l
parted -s /dev/mmcblk0 mkpart extended  12MiB 

```text
root@wengi:/home/hbarta# parted -l
Model: Micron 2450 NVMe 256GB (nvme)
Disk /dev/nvme0n1: 256GB
Sector size (logical/physical): 512B/512B
Partition Table: msdos
Disk Flags: 

Number  Start   End     Size    Type     File system  Flags
 1      4194kB  541MB   537MB   primary  fat32        lba
 2      541MB   43.5GB  42.9GB  primary  ext4
 3      43.5GB  256GB   213GB   primary  zfs


Model: SD SH32G (sd/mmc)
Disk /dev/mmcblk0: 31.9GB
Sector size (logical/physical): 512B/512B
Partition Table: msdos
Disk Flags: 

Number  Start   End     Size    Type     File system  Flags
 1      4194kB  537MB   533MB   primary  fat16        lba
 2      537MB   2621MB  2085MB  primary  ext4


root@wengi:/home/hbarta# 
```

Create extended partition in the middle of free space

```text
parted  /dev/mmcblk0 mkpart extended  12GiB 20GiB
```

Result - boot hang and nothing resized. Delete extended partition and boot - success! Partition table following resizing.

```text
root@wengi:/home/hbarta# echo "unit s  print"|parted /dev/mmcblk0
GNU Parted 3.5
Using /dev/mmcblk0
Welcome to GNU Parted! Type 'help' to view a list of commands.
(parted) unit s  print                                                    
Model: SD SH32G (sd/mmc)
Disk /dev/mmcblk0: 62333952s
Sector size (logical/physical): 512B/512B
Partition Table: msdos
Disk Flags: 

Number  Start     End        Size       Type     File system  Flags
 1      8192s     1048575s   1040384s   primary  fat16        lba
 2      1048576s  62333951s  61285376s  primary  ext4

(parted)                                                                  
root@wengi:/home/hbarta# 
```

Resize root to 10GiB and boot. root expands. Resize again and boot.

Partitions before reboot

```text
Number  Start      End        Size       Type      File system  Flags
 1      8192s      1048575s   1040384s   primary   fat16        lba
 2      1048576s   21528575s  20480000s  primary   ext4
 3      21528576s  62332927s  40804352s  extended
```

And the system hangs. Try the '-' size to create to end of device

```text
parted  -s -- /dev/mmcblk0 mkpart extended  10GiB -1s
```

resulting in

```text
Number  Start      End        Size       Type      File system  Flags
 1      8192s      1048575s   1040384s   primary   fat16        lba
 2      1048576s   9240575s   8192000s   primary   ext4
 3      20971520s  62333951s  41362432s  extended               lba
```

(Switch host to `olive` where the SD comes up as `/dev/sdb`) Also try playbook <https://github.com/HankB/polana-ansible/blob/main/provision-Debian.yml> (using a SATA SSD) and find that this works. It resizes the boot, partition *and* resizes the filesystem and then creates a third partition. Trying that on the SD card:

```text
xzcat /home/hbarta/Downloads/Pi/Debian/20231109_raspi_4_bookworm.img.xz >/dev/sdb # copy image
parted  -s /dev/sdb rm 2     # delete boot partition
parted  -s -- /dev/sdb mkpart primary  1048576s 10GiB # Recreate boot partition
e2fsck -p -f /dev/sdb2 && resize2fs /dev/sdb2 # fsck and resize
echo "unit s  print"|parted /dev/sdb
```

Check partition table

```text
Number  Start     End        Size       Type     File system  Flags
 1      8192s     1048575s   1040384s   primary  fat16        lba
 2      1048576s  20971519s  19922944s  primary  ext4
```

Add extended partition

```text
parted  -s -- /dev/sdb mkpart extended  20971520s -1s
```