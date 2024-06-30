# pmb

Raspberry Pi Multi Boot

Boot multiple operating systems from a single media device (SD, SSD, etc.) The goal is to provide scripts (or Ansible playbooks) to facilitate installing more than one OS on a bootable storage device and allowing the OS to boot from that.

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
