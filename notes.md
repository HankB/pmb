# notes

Exploring the strategy.

## 2024-06-29 boot

* <https://www.raspberrypi.com/documentation/computers/raspberry-pi.html#raspberry-pi-boot-eeprom>
* <https://www.raspberrypi.com/documentation/computers/raspberry-pi.html#eeprom-boot-flow>
* <https://www.raspberrypi.com/documentation/computers/raspberry-pi.html#raspberry-pi-boot-modes>

In the boot FS in `cmdline.txt` the root filesystem is indicated by

```text
console=tty0 console=ttyS1,115200 root=LABEL=RASPIROOT rw fsck.repair=yes net.ifnames=0  rootwait
```

Aside: This has resulted in the wrong root FS benign mounted when multiple media are installed, each with a partitioned named `RASPIROOT`. Note: This is a Debian install. The corresponding line on a RpiOS is

```text
console=serial0,115200 console=tty1 root=PARTUUID=f541a9d1-02 rootfstype=ext4 fsck.repair=yes rootwait quiet splash plymouth.ignore-serial-consoles cfg80211.ieee80211_regdom=US
```

Using `PARTUUID` seems like a better identity for the root partition. Is changing it sufficient or does the initramfs need to be rebuilt. Change `cmdline.txt` is all that is needed on Debian Trixie.

## 2024-06-30 trial

### Platform

* Pi 4B/2GB
* Utility host - RpiOS with desktop on an SD card.
* Target - 520GB Inland SATA SSD.
* Candidate OS - RpiOS, Debian (possibly Ubuntu, Alma, Fedora)

### Plan

1. Install RpiOS (desktop w/out extra apps) on the target SATA SSD (using Imager.)
1. Boot the new install and update/upgrade.
1. Adjust partition size to 40GB and create additional 40GB partitions.
