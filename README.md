# pmb

Raspberry Pi Multi Boot

**Warning Will Robinson! This remains a work in progress and operation of any of these scripts can result in an unbootable host. ** For example, I just wiped the boot partition on my Pi 5 NVME while testing `pmb-swap` and intending to target a USB connected SSD. Fortunately that was fairly easy to restore. But if you are testing this it is preferrable to test on a throw away host or have *good* backups.

This is a work in progress and a consequence is that this README rambles. A lot.

Boot multiple operating systems from a single media device (SD, SSD, etc.) The goal is to provide scripts (or Ansible playbooks) to facilitate installing more than one OS on a bootable storage device and allowing the OS to boot from any of them.

## Status

* Manual swap between two installations (RpiOS and Debian Bookworm) has been demonstrated. [See notes](./notes/notes.md). Added a third partition and Debian Trixie boots w/out problems from a logical partition. Coding pmb-swap is WIP.
* There is a first cut of the script to provision an image for RpiOS `pmb-init`. It uses environment variables `size` (size of root partition) and `device` (target device for configuration.)
* First cut of `pmb-add` is working and has had sone testing.

## TODO

* Add spoofing for Ethernet and Wifi MAC addresses.
* Fix list of mount points to `umount` (`$umount_list`) to work with mount points that may contain spaces.
* Paramaterize boot backup path (`.../root/boot_part/backup.tar`)

### Roadmap

* `pmb-init` - WIP - Start with storage that has been initialized but before first boot and prepare for additional OS installs.
* `pmb-add` - WIP - Allocate space for and add an additional OS, copying from the install image.
* `pmb-swap` - not started - Swap between the current OS and one of the others.
* `pmb-replace` - not started - Replace an existing install (not the active one!) with another OS or fresh install.
* `pmb-copy` - not started - Copy an existing install from other media to the current media.

## Usage:

**Note: This is meant to be run following installation using the Imager and before the first boot**

**Be certain you specify the correct device**

1. Install the OS (RpiOS is likely to be the most tested.)
1. Run as root:

```text
device=/dev/sdb size=10G ./pmb-init 
```

3. Boot into the new installation, install `gparted` (or the tool of yuour choice) to confirm the desired disk partitions. TODO: provide test example.

This script does not (yet) work for Debian. For that see the README in the Ansible directory:

## Theory of operation

The Raspberry Pi initiates the boot process using files in a FAT boot partition. `cmdline.txt` tells the boot process where to find the EXT4 root file system. A drive can have multiple independant root filesystems and by swapping to the corresponding FAT boot partition, the corresponding root filesystem (and hence, installation) can be selected for the next boot.

## Boot process

1. GPU powers up and performs initial H/W init.
1. GPU reads in file(s) from the FAT partition and executes them.
1. Files in the FAT (boot) partition launch the kernel and point to (mount?) the EXT4 (root) partition.
1. Kernel continues to bring the system up.

TODO: Confirm the details of the boot process, in particular how the files in the boot partition link to the respective root partition.

## Plan

There will be three scripts.

1. Initialize the SSD following first installation of RpiOS (Recommended, eventually any OS should work.)

1. Transfer an installation from a normal Pi installation to the multi-boot environment. (e.g. installation.)
1. Swap contents of the FAT partition to the one desired for the subsequent boot. These contents will be stored in the EXT4 partition for the respective OS installation.

The boot media will have a a single FAT/boot partition and separate EXT4/root partitions for each installed OS. The EXT4/root partition will hold a copy of the respective boot partition. To switch to a different OS:

1. Contents of the boot partition will be saved to designated storage in the respective EXT4 partition.
1. Contents of the boot partition will be replaced with contents of the boot partition storage from the EXT4 partition of the target system.
1. System is ready to reboot.

This process will be explored manually to identify the detailed requirements and prove feasability.

## Decisions

I plan to use `bash` shell scripts to implement this. I suspect that Ansible might be better but I don't want to make that a hard requirement for using this tool.

Additional OS partitions will be in extended partitions to facilitate adding more than 2 other OS images.

Tieing IP address to H/W MAC is fraught. Each OS would get the same IP sddress if using DHCP. For this reason it is convenient to provide `.link` files in `/etc/systemd/network` to provide spoofed MAC addresses for both Ethernet and WiFi (based on `Driver` matching.)

The files that will be copied to boot partition (AKA FAT filesystem partition) will be copied to `/root/boot-backup/` and named `install-backup.tar` or `swap-backup.tar`. Compressing these did not seem to save much so that is not done. The difference is because the source directory will be `/boot/firmware` when preparing to swap and `/tmp/<some_dir>/` during initialization or addition and it is not clear if the same command can be used to extract to `/boot/firmware`. This may be a TODO.

When a swap is performed, the `/boot/firmware/cmdline.txt` will be adjusted to select the root partition by `PARTUUID` as revealed by `blkid`. The `/etc/fstab` will be similarly modified to match the present PARTUUIDs for root an boot partitions. (These change when RpiOS first boots.)

Potential swap targets will be identified by the root partition `LABEL`, again as identified by `blkid`. The partition labels will be provided when the OSs are installed (either by `pmb-init` or `pmb-add`) and the partitions will be labeled accordingly. Some OSs select the root partition by label and this leads to issues when there are multiple installs. The boot process may select the wrong root partition and the partition lapel may be a convenient way to identify OS partitions. An alternative would be to record partition IDs and OS names ion a text file somewhere and use that to identify optiosn available when swapping.

For some OSs, it may be useful to configure some things during `pmb-init` or `pmb-add`. (user name, SSH credentials, hostname etc.) The add/init scripts will be modified to support execution of such helpers.

Using partition labels alone seems like a troublesome way to identify OS partitions since a user (me!) will likely create additional partitions and label them. Instead an extra logical partition will be created in the extended partition of nominal size (4GB?) and which will contain a list of the partitions. The partitions information will be stored in a file named to match the partition label. Contents of the partition file will be in the form of ENV variables that the script can source and will include (for example)

```text
"RpiOS" "1aa9757c-02" "dev/mmcblk0p2"
```

It's awkward for distro specific helpers to have to mount partitions to tweak things (and even harder to keep things straight.) For that reason, helpers can expect the following mounts in the indicated environment variables.

* `boot_mnt` - Mount point for the source boot (FAT) partition. This may be a loop mount and in that case will be mounted read/write.
* `root_mount` - Mount point for the root filesystem source, also possibly a loop mount. In the case of `pmb-init` this is not provided as there is no "source" and the "target" is modified in place (and not copied form the source.)
* `target_mnt` - Mount point where the OS is installed and will run from.
* `pmb_system_mnt` - Mount point for auziliary partition which holds the table of installed OSs in `${pmb_system_mnt}/installed-OS`. If a particular OS needs additional information it can be stored here (in an OS specific file.)

In addition, the following variables will provide the UUID for boot and root partitions.

* `partuuid`
* `bootuuid`

There is an explicit assumption that all OS partitions and the single boot partition reside on the same storage device for two reasons:

* make it easier to ID 'from' and 'to' partitions when swapping. choosing ther wrong partition will have bad results.
* It makes little sense to have boot and root partitions on different devices (at this point in time. Feel free to convince me otherwise.)

## Details

### Initial provisioning

On first boot, all OS images of which I am aware expand the EXT4 partition to use the remainder of the storage medium. This can be constrained by creating an additional disk partition that limits the space available for expansion to the empty space between the EXT4 partition and the additional partition.

The Pi uses MBR partitions so in order to provide partitions for more than two additional OS images some will need to be extended partitions.

It will be necessary to verify that the Pi can use an EXT4 partition in an extended partition. (The boot partition will remain in a primary partition as it is reused among all OSs.)

The first task is to create a pre-first boot playbook that will:

1. Add the "blocker" partition to limit expansion of the original EXT4 partition. (In the case of Debian it is necessary to create a partition at the end of the Extended partition.)
1. Create the files needed to spoof Ethernet and WiFi MAC addresses. (This is a convenience to allow the DHCP server to assign each OS a unique IP address.)

Parameters include:

* Target device
* Size for root partition.
* (Eventually) Ethernet and WiFi MAC address to spoof.

### Adding another OS

This will be allocated to a Logical partition in the Extended partition.

Details:

* This is best performed when booted from media other than the target as formatting programs get cranky about modifying the partition table when partitions are mounted and active. (Probably boot from SD and modify SATA/USB or NVME SSD.)
* Identify the starting sector of the next partition.
* Create and format and mount the partition.
* Uncompress the source image to `/tmp`
* Identify the partitions inside the image and mount thep (e.g. `losetup`)
* Copy the root partition to the new added partition.
* "Backup" the boot (FAT) partition to the new partition as was done when swapping
OSs. This eliminates the need to identify which OS was previously active and back it up before copying the new boot sector in place.

### Swapping to another OS.

* Identify candidates by label as shown by `lsblk`. Q: How will this differentiate between OS installations and other partitions the user may create? It seems like it will be necessary to keep a table of OS installations and their partitions, perhaps in the original OS installation filesystem. Or does it make sense to create a small filesystem to include this? (Yes, small filesystem to manage this.)
* Confirm settings in `/boot/firmware/cmdline.txt` and `/etc/fstab` to make certain they have not been modified by an upgrade since originally performed.

## Alternatives

* <https://github.com/procount/pinn> PINN is a more mature and polished boot selector. It is actively supported and works on a Pi 5. Specific advantages include:

    * Point and shoot selection of candidate OSs.
    * Select OS during the boot process.
    * Graphical interface suitable for any level of expertise.
    * Apparent availability of custom images.

Compared to what I am working on, `pmb` will likely require more expertise to leverage but should work with any Pi OS that uses the FAT/EXT4 boot/root filesystem layout. AFAIK the way the Pi boots constrains the file layout this way.

## Errata

* The `bash` scripts will be `shellcheck` clean and include some boilerplate from a site that no longer exists. (https://kvz.io/bash-best-practices.html or https://github.com/kvz/bash3boilerplate.)
* All UUIDs are treated as case sensitive but technically they may not be. They represent an integer value and representation as a hex string is a convention.
