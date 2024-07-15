# testing notes

First component, first cut complete. Now to exercise in different settings.

## 2024-07-12 back to bash

Performing some testing with `pmb-init` and some other distros including Ubuntu, Alma and also using a variety of media including SD card, USB/SSD and perhaps NVME SSD.

### RpiOS

Use imager to load OS and configure. SD and SATA SSD work.

## 2024-07-13 focus on other OS

Platforms:

* X86_64 laptop with SD card slot and USB ports
* CM4 using SD card or NVME adapter.
* Pi 4B, SD and SATA SSD.

### Almalinux

Initial image had 2 partitions, 1 and 9. After first boot there were 1 and 2.

No keyboard. No mouse. No go.

### Ubuntu 22.04 desktop and server

Installed with `rpi-imager` and no customizations. Booted OK and went through startup initializations and then got an "install failed" reporting issues with `sddm`. Got a login screen but could not log in. Bad password? Tried in a text console still no joy. Gried `<ctrl><alt><del>` whih was delayed due to `unattended-upgrades` and finally came up to RpiOS (from SD card.)

Repeating the install using server to sidestep possible desktop issue. Process with `pmb-init` and it boots w/out issue.

### Fedora

PITA - seems like the necessary information is scattered around a bunch of pages. Just going to download an image and blast it to NVME SSD. It shows up with three partitions including P3 which is btrfs. Booting it to see what happens. This:

```text
U-boot>
```

Maybe SD card. Yes, Fedora 40 boots and runs from an SD card but that is considered not interesting. Did't try a USB/SSD. It did not boot from SSD in a CMR/IO Board with PCIe/NVME SSD.
