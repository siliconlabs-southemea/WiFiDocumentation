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
2023-05-03:
  * 64-bit Mathematica added to rp-prefapps
  * Bug fix - occasional segfault in CPU temperature plugin
  * Bug fix - X server crash when changing screen orientation
  * Bug fix - X server DPMS not working
  * Mathematica updated to 13.2.1
  * Matlab updated to 23.1.0
  * Chromium updated to 113.0.5672.59
  * Raspberry Pi Imager updated to 1.7.4
  * RealVNC server updated to 7.0.1.49073
  * RealVNC viewer updated to 7.0.1.48981
  * Updated VLC HW acceleration patch
  * libcamera
    - Add generalised statistics handling.
    - Fix overflow that would cause incorrect calculations in the AGC algorithm.
    - Improve IMX296 sensor tuning.
  * libcamera-apps
    - Improve handling of audio resampling and encoding using libav
    - Improve performance of QT preview window rendering
    - Add support for 16-bit Bayer in the DNG writer
    - Fix for encoder lockup when framerate is set to 0
    - Improved thumbnail rendering
  * picamera2
    - MJPEG server example that uses the hardware MJPEG encoder.
    - Example showing preview from two cameras in a single Qt app.
    - H264 encoder accepts frame time interval for SPS headers.
    - H264 encoder should advertise correct profile/level.
    - H264 encoder supports constant quality parameter.
    - Exif DateTime and DateTimeOriginal tags are now added.
    - Various bug fixes (check Picamera2 release notes for more details).
  * Some translations added
  * Raspberry Pi firmware 055e044d5359ded1aacc5a17a8e35365373d0b8b
  * Linux kernel 6.1.21
```

### On an already running RaspiOS image

Retrieve the linux kernel version by running `cat` on `/lib/modules/$(uname -r)/build/include/config/kernel.release` :

``` console 
pi@raspberrypi:~ $ cat /lib/modules/$(uname -r)/build/include/config/kernel.release
6.1.21-v8+
```

In this example, our kernel version is `6.1.21`

If this command does not work, fetch running kernel headers as described below

## Fetching kernel sources and headers

### Kernel Headers
This part is the most difficult one to document as it may end up with different results than the below, depending on the RaspiOS version

Finding a unique way that works for all was not possible, therefore I will detail one and provide others in the *Additional Info* section below 

Linux headers are usually provided with any distribution. But are also available through apt packages.

Since recently, raspberry headers can be found via the below apt package :

``` console 
apt install raspberrypi-kernel-headers
```

This should install the latest headers available in apt in /lib/modules

``` console
pi@raspberrypi:~ $ ls /lib/modules
6.1.21-v8+
```

### Kernel sources

Once headers are downloaded and that we know our version we can get the kernel sources. 

Before anything though, prepare a workspace :

``` console
cd ~
mkdir wfx_driver 
cd wfx_driver
```

It turns out that Raspberry's Linux git repo does not bring in the latest WFX driver. This makes it more complex to build the WFx kernel module on that platform. 

Silicon Labs provides latest updates in the [linux upstream kernel](https://elixir.bootlin.com/linux/v6.6.6/source/drivers/net/wireless/silabs/wfx) branch

Therefore it is easier to fetch sources directly from there. Easiest is to directly clone the branch matching your kernel; version :

``` console
git clone --depth=1 git://git.kernel.org/pub/scm/linux/kernel/git/stable/linux-stable.git -b v6.1.21
```

Do not forget to append `v` to the branch name : `v6.1.21`

## Additional info

### Fetching sources 
Another way of getting specific Kernel sources from thye officia git repo branches has been developed and made available [here](https://github.com/RPi-Distro/rpi-source)

Not tested as the latest images 

Also, linux headers may have been previously retrieved using `linux-headers-generic linux-image-generic`

Unfortunately 