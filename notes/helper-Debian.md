# Debian helper

See <https://raspi.debian.net/defaults-and-settings/> for details.  The problem: for `pmb-init` the boot filesystem is mounted and `/boot/firmware/sysconf.txt` can be created in place. For `pmb-add`, the destination is not mounted but rather added as a tar archive. It will be necessary to determine which script sourced thiws helper.

Ah... The sourced script still reports the path to the script which sourced it iun `$0`. That's helpful.
