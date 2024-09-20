---
sort: 1
---
# Getting Kernel Sources - RaspiOS users

## Identifying the currently used kernel version

### Prior to downloading the RaspiOS image

Go to the official Raspberry Pi Image [release page](https://www.raspberrypi.com/software/operating-systems/)

There you will be able to retrieve the changelog tied to the OS Image you chose using Raspberry Pi Imager. You will also be provided with the image release date

Go to the [RaspiOS Changelog](https://downloads.raspberrypi.com/raspios_armhf/release_notes.txt) and check which datecode which was released the soonest. This is the one that Raspberry Pi Imager will fetch

At this point you will be able to get the proper kernel sources directly from the linux-stable kernel repo, branch matching your kernel. For example :

```
2024-07-04:
  * pipanel - allow customisation of more than 2 desktops
  * pipanel - add customisation for labwc
  * gui-pkinst - add whitelist to restrict installation to specified packages only
  * pixflat-theme - add theme settings for labwc
  * pishutdown - revert to original use of pkill to close desktop
  * piclone - fix for potential buffer overflow vulnerability (that would never have actually happenedâ€¦)
  * lp-connection-editor - fix dialog icons on taskbar
  * rp-prefapps - add Raspberry Pi Connect; remove SmartSim
  * piwiz - add page to enable / disable Raspberry Pi Connect
  * wf-panel-pi - constrain main menu to fit on small screens
  * wf-panel-pi - fix dialog icons on taskbar
  * wf-panel-pi - fix keyboard handling and icon highlighting for taskbar buttons
  * raspberrypi-ui-mods - add configuration for labwc
  * raspberrypi-ui-mods - add support for new touchscreens
  * raspberrypi-ui-mods - systemd-inhibit used to override hardware power key on Pi 5
  * rc-gui - add configuration of alternate keyboard layout
  * rc-gui - add switching for Raspberry Pi Connect
  * arandr - add brightness control for DSI displays
  * arandr - more reliable method to detect virtual displays
  * raspi-config - add setting of keyboard options
  * raspi-config - add setting of PCIe speed
  * raspi-config - add switching for Raspberry Pi Connect
  * wayvnc - better handling for virtual displays
  * wayvnc - improved encryption support
  * GTK-3 - add keyboard shortcuts in combo boxes
  * pcmanfm - allow customisation of more than 2 desktops
  * pcmanfm - fix bug causing crash and inconsistent behaviour on certain drag and drop operations
  * raspberrypi-sys-mods - add udev rule to allow backlight change
  * raspberrypi-sys-mods - increase swapfile size
  * raspberrypi-sys-mods - remove symlinks from paths in initramfs scripts
  * wayfire - fix for crash when opening multiple Xwayland windows
  * wayfire - fix for touchscreen bug when touching areas without windows
  * labwc compositor installed as an alternative to wayfire; can be enabled in raspi-config
  * various small bug fixes and tweaks
  * Chromium updated to 125.0.6422.133
  * Firefox updated to 126.0
  * Raspberry Pi firmware 3590de0c181d433af368a95f15bc480bdaff8b47
  * Linux kernel 6.6.31 - c1432b4bae5b6582f4d32ba381459f33c34d1424
```

### On an already running RaspiOS image

Retrieve the linux kernel version by running `cat` on `/lib/modules/$(uname -r)/build/include/config/kernel.release` :

```console
pi@raspberrypi:~ $ cat /lib/modules/$(uname -r)/build/include/config/kernel.release
6.6.31+rpt-rpi-v8
```

In this example, our kernel version is `6.6.31`

If this command does not work, fetch running kernel headers as described below

## Fetching kernel sources and headers

### Kernel Headers

This part is the most difficult one to document as it may end up with different results than the below, depending on the RaspiOS version

Finding a unique way that works for all was not possible, therefore I will detail one and provide others in the *Additional Info* section below

Linux headers are usually provided with any distribution. But are also available through apt packages.

Since recently, raspberry headers should be found in /lib/modules :

```console
pi@raspberrypi:~ $ ls /lib/modules
6.6.31+rpt-rpi-2712  6.6.31+rpt-rpi-v8
```

If not, via the below apt package :

```console
apt install raspberrypi-kernel-headers
```

This should install the latest headers available in apt in `/lib/modules`. Verify it matches the `uname -r` version.

### Kernel sources

Once headers are downloaded and that we know our version we can get the kernel sources.

Before anything though, prepare a workspace :

```console
cd ~
mkdir wfx_driver 
cd wfx_driver
```

It turns out that Raspberry's Linux git repo does not bring in the latest WFX driver. This makes it more complex to build the WFx kernel module on that platform.

Silicon Labs provides latest updates in the [linux upstream kernel](https://elixir.bootlin.com/linux/v6.6.6/source/drivers/net/wireless/silabs/wfx) branch

Therefore it is easier to fetch sources directly from there. Easiest is to directly clone the branch matching your kernel; version :

```console
git clone --depth=1 git://git.kernel.org/pub/scm/linux/kernel/git/stable/linux-stable.git -b v6.6.31
```

Do not forget to append `v` to the branch name : `v6.6.31`

## Additional info

### Fetching sources

Another way of getting specific Kernel sources from thye officia git repo branches has been developed and made available [here](https://github.com/RPi-Distro/rpi-source)

Not tested as the latest images

Also, linux headers may have been previously retrieved using `linux-headers-generic linux-image-generic`

Unfortunately
