# pmb

Raspberry Pi Multi Boot

Boot multiple operating systems from a single media device (SD, SSD, etc.) The goal is to provide scripts (or Ansible playbooks) to facilitate installing more than one OS on a bootable storage device and allowing the OS to boot from that.

## Status

Manual swap between two installations (RpiOS and Debian Bookworm) has been demonstrated. [See notes](./notes.md). Added a third partition and Debian Trixie boots w/out problems from a logical partition.

There is a first cut of the script to provision an image for RpiOS `pmb-init`. It uses environment variables `size` (size of root partition) and `device` (target device for configuration.)

**Note: This is meant to be run following installation using the Imager and before the first boot**

Usage (following installation using the Imager):

```text
device=/dev/sdb size=10G ./pmb-init 
```

This script does not (yet) work for Debian. For that an Ansible playbook is provided and its usage is:

```text
cd Ansible
/choose/editor pmb_init_vars.yml # and tauilor for your preferences
ansible-playbook provision-Debian-lite.yml -b -K
```

This will install and tailor the image. Installing a Debian image using the Imager results in a system that does not boot.

## Theory of operation

The Raspberry Pi initiates the boot process using files in a FAT boot partition. `cmdline.txt` tells the boot process where to find the EXT4 root file system. A drive can have multiple independant root filesystems and by swapping to the corresponding FAT boot partition, the corresponding root filesystem (and hence, installation) can be selected for the next boot.

## Boot process

1. GPU powers up and performs initial H/W init.
1. GPU reads in file(s) from the FAT partition and executes them.
1. Files in the FAT (boot) partition launch the kernel and point to (mount?) the EXT4 (root) partition.
1. Kernel continues to bring the system up.

TODO: Confirm the details of the boot process, in particular how the files in the boot partition link to the respective root partition.

## Plan

There will be two processes.

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

## Details

### Initial provisioning

On first boot, all OS images of which I am aware expand the EXT4 partition to use the remainder of the storage medium. This can be constrained by creating an additional disk partition that limits the space available for expansion to the empty space between the EXT4 partition and the additional partition.

The Pi uses MPR partitions so in order to provide partitions for more than two additional OS images some will need to be extended partitions.

 it will be necessary to verify that the Pi can use an EXT4 partition in an exgtended partition. (The boot partition will remain in a primary partitiion as it is shared among all OSs.)

The first task is to create a pre-first boot playbook that will:

1. Add the "blocker" partition to limit expansion of the original EXT4 partition.
1. Create the files needed to spoof Ethernet and WiFi MAC addresses. (This is a convenience to allow the DHCP server to assign each OS a unique IP address.)

Parameters include:

* Target device

## Alternatives

* <https://github.com/procount/pinn> PINN is a more mature and polished boot selector. It is actively supported and works on a Pi 5. Specific advantages include:

    * Point and shoot selection of candidate OSs.
    * Select OS during the boot process.
    * Graphical interface suitable for any level of expertise.
    * Apparent availability of custom images.

Compared to what I am working on, `pmb` will likely require more expertise to leverage but should work with any Pi OS that uses the FAT/EXT4 boot/root filesystem layout. AFAIK the way the Pi boots constrains the file layout this way.
