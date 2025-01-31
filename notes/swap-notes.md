# swap notes

## 2024-08-09 manual swap

[from add-notes](./add-notes.md#2024-08-09-im-back)

Time to codify the policy on where the `/boot` (AKA `FAT`) partition is stored. And oops, `pmb-add` does not save the boot partition. There is more work to to there. Back to `add-notes.md`

## 2024-08-21 manual swap

There was a lot more work for `pmb-add` but that looks to be complete. At present have an SSD connected to a Pi 4B to test swap with the intent to validate `pmb-add`

1. Mount new root partition

```text
mkdir /mnt/root
mount /dev/sda6 /mnt/root
cd
mkdir boot-backup
cd $_
tar cf swap-backup.tar /boot/firmware/
tar tf swap-backup.tar
cd /mnt/root/root/boot-backup/
tar -tvf install-backup.tar --strip-components=2
tar -xf --strip-components=2 /mnt/root/root/boot-backup/install-backup.tar

rm -r /boot/firmware/*
ls -l /boot/firmware
tar --strip-components=2 --directory=/boot/firmware -xf /mnt/root/root/boot-backup/install-backup.tar
```

And reboot. No joy. Examine boot part and find the wrong `PARTUUID="a3f161f3-06"`

```text
/dev/sda6: LABEL="Debian12" UUID="0d732d9b-53d5-4dc7-9b18-20f2a8283677" BLOCK_SIZE="4096" TYPE="ext4" PARTUUID="e3e8356d-06"
```

And so began a cluster-F that included:

* Unable to unmount the boot partition on the SATRA SSD.
* Hangs rebooting
* Somehow booting from the SATA SSD after reasing it in `olive`
* Trying to boot/install from the network after rebooting.
* Hanging on boot
* Finally booting after power cycling.

This system has been a general PITA with the desktop hanging repeatedly when swapping between Pi 5 and Pi 4B with the KVM switch. Will continue testing as much as possibnle from `olive` to avoid the aggravation.

1. secure erase and validate (oak)
1. Install RpiOS using `rpi-imager`
1. `pmb-init vars_RpiOS`
1. `pmb-add vars_Debian_bookworm`

```text
root@wengi:/home/hbarta/Programming/pmb# blkid|grep sda
/dev/sda1: LABEL_FATBOOT="bootfs" LABEL="bootfs" UUID="44FC-6CF2" BLOCK_SIZE="512" TYPE="vfat" PARTUUID="fb33757d-01"
/dev/sda2: LABEL="RpiOS" UUID="93c89e92-8f2e-4522-ad32-68faed883d2f" BLOCK_SIZE="4096" TYPE="ext4" PARTUUID="fb33757d-02"
/dev/sda5: LABEL="pmb-system" UUID="eab77265-4e6a-4b6b-af0d-369599bbd73f" BLOCK_SIZE="4096" TYPE="ext4" PARTUUID="fb33757d-05"
/dev/sda6: LABEL="Debian12" UUID="2c3fdf56-1846-44bd-9263-9964efa3019d" BLOCK_SIZE="4096" TYPE="ext4" PARTUUID="fb33757d-06"
/dev/sda7: PARTUUID="fb33757d-07"
root@wengi:/home/hbarta/Programming/pmb# 
```

```text
root@wengi:/media/hbarta/Debian12# cd root/boot-backup/
root@wengi:/media/hbarta/Debian12/root/boot-backup# tar xf install-backup.tar 
root@wengi:/media/hbarta/Debian12/root/boot-backup# cat tmp/39036769545/sysconf.txt 
hostname=pmb-deb
root_authorized_key=ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQDKU/XY+OQMMvCUXPpOeLlyEdUd5ppIBVFMCxUXOlM120O/VdeBc4B8ckqCZn2hdXshMbLgQV1+Dd8ZI79EZnvh4lICJ+3NTKtBVIeAPN1VLjNF+rnzxt+OtDjjCyJFmApvjTxC3D0UkyV4wtCwgbOl/r8NNJJonFOdcrsPDbstL4ASZU4pETLoA1xU+o2cxn0Z7erZ8uqSjBBrTTr7U2aJB+H8V5gh70OupaH8D6pCBm50Vqdza/AwonCOoSgGQuamUhD1urT/Nt9P3k5XwFRcQk3j8aNenuo3x0IBZ1G1hlGmWSNW6kh+1CWwU4/sipJvzycJHS47/eybGOqtTTxYKvQmLVHlCUkV5kQdKOZXmZQRsw6NzSDon56kNnvBrS2jcHTUT5CUIGM/sN7idmNdzblWqIn7nDoqZUCVB3tMq1l5O9vUym7in2YuGmlRAChLB3zXS1kxQw3/raUUdV7mZIN8TEgzXyv4sRXrNgUZ2avAXTl6XNKvEepmj3YNbrxcHAmNdm22emFtNO8XtQqMKrVI/ulB92I01DFBAvxbJb8rRt9CVa4WKd4gIMxjPi+L/uqV416cPChJLUFQbXCyohnPtJ/kkW1fe/t/yvoisZehpnItuEFlIyN7rMf+KGPyclo9QvgU5CrHDQN3kZ5ayBj0iLurxNsHR8JZlftDbw== hbarta@gmail.com
root@wengi:/media/hbarta/Debian12/root/boot-backup# cat tmp/39036769545/cmdline.txt 
console=tty0 console=ttyS1,115200 root=PARTUUID="fb33757d-06" rw fsck.repair=yes net.ifnames=0 rootwait 
root@wengi:/media/hbarta/Debian12/root/boot-backup# blkid | grep fb33757d-06
/dev/sda6: LABEL="Debian12" UUID="2c3fdf56-1846-44bd-9263-9964efa3019d" BLOCK_SIZE="4096" TYPE="ext4" PARTUUID="fb33757d-06"
root@wengi:/media/hbarta/Debian12/root/boot-backup# 
```

All good so far. Now boot the installation.

```text
root@pmb-rpios:/home/hbarta# blkid
/dev/sda2: LABEL="RpiOS" UUID="93c89e92-8f2e-4522-ad32-68faed883d2f" BLOCK_SIZE="4096" TYPE="ext4" PARTUUID="c44561c7-02"
/dev/sda5: LABEL="pmb-system" UUID="eab77265-4e6a-4b6b-af0d-369599bbd73f" BLOCK_SIZE="4096" TYPE="ext4" PARTUUID="c44561c7-05"
/dev/sda1: LABEL_FATBOOT="bootfs" LABEL="bootfs" UUID="44FC-6CF2" BLOCK_SIZE="512" TYPE="vfat" PARTUUID="c44561c7-01"
/dev/sda6: LABEL="Debian12" UUID="2c3fdf56-1846-44bd-9263-9964efa3019d" BLOCK_SIZE="4096" TYPE="ext4" PARTUUID="c44561c7-06"
/dev/sda7: PARTUUID="c44561c7-07"
root@pmb-rpios:/home/hbarta# blkid | grep fb33757d-06
root@pmb-rpios:/home/hbarta# 
```

PARTUUID has changed! This needs a fixup when swapping. That was probably always needed anyway. Repeating the swap steps and fixing `cmdline.txt` and `raspi-firmware`. And boot. Hung because Ethernet not connected. Connected and sort of came up, but networking not fully configured and boot partition not mounted. Rebooting but seems not good. Ah... Did I fix the boot partition mount in `/etc/fstab`. No, I did not. Got a clean boot once the format for `/etc/fstab` was correct. Yay!

## 2024-08-22 test latest changes

Backup boot partition.

```text
mkdir -p /root/boot-backup
tar cf /root/boot-backup/swap-backup.tar /boot/firmware/
tar tf /root/boot-backup/swap-backup.tar
```

Replace boot partition with other OS'

```text
mkdir /mnt/root
mount /dev/sda6 /mnt/root
rm -r /boot/firmware/*
tar --strip-components=2 --directory=/boot/firmware -xf /mnt/root/root/boot-backup/install-backup.tar
ls -l /boot/firmware
```

Looks right. Now check `cmdline.txt` and `/etc/fstab` in the next OS.

```text
cat /boot/firmware/cmdline.txt
cat /mnt/root/etc/fstab
```

```text
root@pmb-rpios:~# cat /boot/firmware/cmdline.txt
console=tty0 console=ttyS1,115200 root=PARTUUID="a3f161f3-06" rw fsck.repair=yes net.ifnames=0 rootwait 
root@pmb-rpios:~# cat /mnt/root/etc/fstab
# The root file system has fs_passno=1 as per fstab(5) for automatic fsck.
PARTUUID="a3f161f3-06" / ext4 rw 0 1
# All other file systems have fs_passno=2 as per fstab(5) for automatic fsck.
PARTUUID="a3f161f3-01" /boot/firmware vfat rw 0 2
root@pmb-rpios:~# blkid|grep a3f161f3
root@pmb-rpios:~# blkid
/dev/sda2: LABEL="RpiOS" UUID="56f80fa2-e005-4cca-86e6-19da1069914d" BLOCK_SIZE="4096" TYPE="ext4" PARTUUID="eae80a8e-02"
/dev/sda5: LABEL="pmb-system" UUID="72aec940-6a3f-4457-a03e-194bc7edcb92" BLOCK_SIZE="4096" TYPE="ext4" PARTUUID="eae80a8e-05"
/dev/sda1: LABEL_FATBOOT="bootfs" LABEL="bootfs" UUID="91FE-7499" BLOCK_SIZE="512" TYPE="vfat" PARTUUID="eae80a8e-01"
/dev/sda6: LABEL="Debian12" UUID="b35742f8-0abe-48aa-a9a4-fb78b5b93cbd" BLOCK_SIZE="4096" TYPE="ext4" PARTUUID="eae80a8e-06"
/dev/sda7: PARTUUID="eae80a8e-07"
root@pmb-rpios:~# 
```

Fixup PARTUUID

```text
root@pmb-rpios:~# sed -i s/a3f161f3/eae80a8e/ /boot/firmware/cmdline.txt
root@pmb-rpios:~# sed -i s/a3f161f3/eae80a8e/ /mnt/root/etc/fstab
root@pmb-rpios:~# grep eae80a8e /mnt/root/etc/fstab /boot/firmware/cmdline.txt
/mnt/root/etc/fstab:PARTUUID="eae80a8e-06" / ext4 rw 0 1
/mnt/root/etc/fstab:PARTUUID="eae80a8e-01" /boot/firmware vfat rw 0 2
/boot/firmware/cmdline.txt:console=tty0 console=ttyS1,115200 root=PARTUUID="eae80a8e-06" rw fsck.repair=yes net.ifnames=0 rootwait 
root@pmb-rpios:~# 
```

And reboot - Great Success! Just remember to reconnect the Ethernet cable. Reboot - no joy. Did the blkid change again? No, forgot to change `/media/hbarta/Debian12/etc/default/raspi-firmware`. Need to add:

```text
sed -i s/a3f161f3/eae80a8e/ /mnt/root/etc/default/raspi-firmware
```

Reboot again - Great Success again! Now swap back to RpiOS.

```text
systemctl daemon-reload
mkdir -p /mnt/root
mount /dev/sda2 /mnt/root
rm -r /boot/firmware/*
tar --strip-components=2 --directory=/boot/firmware -xf /mnt/root/root/boot-backup/swap-backup.tar
ls -l /boot/firmware
```

`cmdline.txt` and `fstab` look good. Reboot! Good - but no Ethernet, just WiFi. Hmmm.

## 2024-08-23 device ID

This is a little sticky. `/dev/sdX` style identifiers will change. `PARTUUID` should have been good but RpiOS changes them on first boot. Other OSs could also do that, particularly the ones based on RpiOS. Hopefully the partition labels will stick. That's been the case with testing with Debian and RpiOS so far. Scripts will use that to identify the root device when swapping. And the stored information will also include the specific OS helper.

## 2024-08-25 swap requirements

1. Confirm the desired target is not the current running OS
1. Mount the information partition (label `pmb-system`) and import variables for the desired target, confirming that they exist.
1. Fixup the information file (`fixup_OS_info()`).
1. Mount the target partition and confirm that the boot partition backup exists.
1. Back up the boot partition for the current OS.
1. Delete files from boot partition.
1. Restore files from next OS boot backup.
1. Check `/etc/fstab`, `/boot/firmware/cmdline.txt` and (for Debian) that `/etc/default/raspi-firmware` are all correct (e.g. point to the correct root partition.)
1. Reboot - It seems sensible to reboot automatically at this point rather than leave preparatiosn in plade and leave old OS running. Likewise if there is any problem encountered during preparation, the system should be left as it was with the OS that was already running.

Possible issues - conflict with automated-upgrades (or manual upgrades)/. Possible locking solution <https://old.reddit.com/r/debian/comments/226suc/how_to_properly_get_and_release_the_dpkg_lock/> and <https://lists.debian.org/debian-dpkg/2010/02/msg00054.html>. See also <https://www.baeldung.com/linux/file-locking> for a possible C solution. Or directly in bash <https://stackoverflow.com/questions/66380930/how-to-acquire-a-lock-file-in-linux-bash> and <https://man7.org/linux/man-pages/man1/flock.1.html>

## 2024-09-17 RpiOS mounts other partitions

This is a new wrinkle for `pmb-swap`. With the full desktop installed, RpiOS will mount the info and root partitions for the other installs. Just more stuff for the script to manage. (`pmb-add` as well, I suppose.)
