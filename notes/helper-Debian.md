# Debian helper

See <https://raspi.debian.net/defaults-and-settings/> for details.  The problem: for `pmb-init` the boot filesystem is mounted and `/boot/firmware/sysconf.txt` can be created in place. For `pmb-add`, the destination is not mounted but rather added as a tar archive. It will be necessary to determine which script sourced this helper.

Ah... The sourced script still reports the path to the script which sourced it in `$0`. That's helpful.

Then the strategy becomes

1. create `sysconf.txt` in `/tmp`
1. add `root_authorized_key` (if provided)
1. add `hostname` (if provided)
1. move `sysconf.txt` to boot partition (`pmb-init`) or add it to the tar archive `/root/boot-backup/install-backup.tar` in the new root partition (`pmb-add`).
1. Tweak (what will eventually become)  `/boot/firmware/cmdline.txt` and `/etc/default/raspi-firmware` to get the root device properly specified. (`pmb-init`, `pmb-add`)

Note: `pmb-init` does not mount boot/root partitions so it will be necessary to mount/umount the boot partition. Maybe it should. `pmb-init` will need to mount the root partition to spoof MAC addresses.
