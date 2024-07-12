# Ansible playbooks

These were not the desired method for "scripting" this operation but getting partitioning for Debian working was troublesome and Ansible Playbooks had some advantages.

1. I already had a playbook that did some partition and which worked with Debian.
1. Testing and Debugging is faster using playbooks rather than repeating a lot of shell commands.
1. The playbooks may support more features than the correspinding shell scripts.

I do plan to implement the partitioning strategy in `pmb-init`.

* `pmb_init-Debian.yml` This playbook performs
    * Unmount partitions on the device if mounted.
    * Erase the target device using `blkdiscard` if supported.
    * Installation of a Debian OS image.
    * Expand the root partition.
    * Create an extended partition to hold logical partitions for future installations.
    * Create a small logical partition at the end of the extended partition to allow Debian to complete the boot process.
    * Set a hostname.
    * Set WiFi and Ethernet spoofed MAC addresses using Systemd Link files.
    * Copy SSH credentials for root SSH login.

Parameters are sourced from [`pmb_init_vars.yml`](./pmb_init_vars.yml) and according to the following notes:

* If the `os_image` variable is commented out, the OS image will not be written to the card. This is useful for debugging other parts of the playbook once the image has been written.
* units for `root_size` are according to what `parted` understands.

Typical usage would be:

```text
ansible-playbook provision-Debian-lite.yml -b -K
```
