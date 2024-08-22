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
