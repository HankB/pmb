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
