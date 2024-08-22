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
