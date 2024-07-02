# Explore PINN

This is a much more polished alternative, but it seems to require that distros be prepared to run on PINN. Installing several on the 128GB SD card at present (on a Pi 5).

## 2024-07-01 Pi 5

Installed to 128GB SSD w/ no issue and added three OSs, RpiOS, MX and kde3lee64.

* kde3lee64 boot loops. Now it's coming up. It has a very pretty desktop/theme but still uses Plasma 5.27. System monitor does not work. It looks to be just a themed variant of RpiOS. Hangs on shutdown.
* RpiOS comes up and seems to work normally including mounting the partitions for the other OSs. Speaking of partitions, for 3 OSs there are 11 partitions. It appears that each OS has it's own boot partition and there are extra 2 MiB partitions between each other. And I just lost the mouse. Fresh batteries brought it back. In the mean time PINN auto-booted RpiOS. All installs identify (`/etc/issue`) as `Debian GNU/Linux 12`. Hangs on shutdown.
* MX has a nice first boot configurator (and comes up with typical US settings.)

## 2024-07-02 Pi CM4

Installed to the 1TB 670p NVME SSD and after booting, there was no keyboard or mouse. Rebooted - no change. Tried the older version. Same result. PINN is a no go on the CM4.

## 2024-07-02 more Pi 5

Reminder, running from the SD card. MX looks fairly polished so testing that a bit more. Upgrading ~250 packages before installing KDE. Found an article at <https://www.fosslinux.com/60444/how-to-install-kde-on-mx-linux.htm> and will follow that. Kde comes up but initial trial is not too favorable. I clicked the desktop icon for "MX RPI Introductory Video" and it opened a Konqueror page with a Youtube URL and then just sat there. I launched Chromium and opened the URL. It plays well with low framd droppage. I paired the Oontz speaker and that paired, but there is no audio, either via Youtube or the settings "Test." Also the Status monitor does not work as hoped. It seems that KDE/Plasma on MX is not improved over RpiOS.

MX is worth watching but I'm not sure that PINN provides any benefit for me vs. a home grown OS switcher. Not working on a CM4 is a show stopper there and there ae not that many options listed for Pi 5 support. I was hoping to see Alma but that's a no show on the Pi 5. FAQ is in `/usr/local/share/doc`. And now BT audio is working after reconnecting the speaker. And the MX variant doesn't hang on shutdown.
