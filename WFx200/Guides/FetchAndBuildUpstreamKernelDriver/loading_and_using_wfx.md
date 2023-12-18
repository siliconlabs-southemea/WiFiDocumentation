---
sort: 3
---

# Loading and using the WFx Kernel Module

At this point you should have :
 - Downloaded Kernel headers matching your running platform
 - Downloaded Kernel sources matching your running platform
 - Built the WFX kernel module `wfx.ko`
 - Installed the WFK kernel module on the system

## Loading the kernel module

This step is straightforward if all previously happened successfully . Simply used `modprobe wfx` as super user:

``` console 
pi@raspberrypi:~/wfx_driver/linux-stable/drivers/net/wireless/silabs/wfx $ sudo modprobe wfx
```

To verify the module is loaded just use lsmod this way : `lsmod | grep wfx`

Output should be :

``` console 
pi@raspberrypi:~/wfx_driver/linux-stable/drivers/net/wireless/silabs/wfx $ lsmod | grep wfx
wfx                   151552  0
mac80211              974848  1 wfx
cfg80211              925696  3 wfx,brcmfmac,mac80211
```

## Using the kernel module to enable the WF200 :

### Enabling module load at boot time

This step is distribution dependent. As we are running RaspiOS, we will be using systemd and its ability to load mosules at boot time.

Edit `/etc/modules-load.d/modules.conf` as super user and add `wfx` at the end of the file as below:

``` console 
pi@raspberrypi:~ $ cat /etc/modules-load.d/modules.conf
# /etc/modules: kernel modules to load at boot time.
#
# This file contains the names of kernel modules that should be loaded
# at boot time, one per line. Lines beginning with "#" are ignored.

i2c-dev
wfx
```

Once done just reboot : `sudo reboot`

Just check the module is now loaded upon startup by calling `lsmod | grep wfx` :

``` console 
pi@raspberrypi:~ $ lsmod | grep wfx
wfx                   151552  0
mac80211              974848  1 wfx
cfg80211              925696  2 wfx,mac80211
```

You can check if the driver is detecting the card properly by using `dmesg | grep wfx` :

``` console
pi@raspberrypi:~ $ dmesg | grep wfx
[    6.687705] wfx: loading out-of-tree module taints kernel.
[    6.740432] wfx-sdio mmc1:37dd:1: no compatible device found in DT
```

### Disabling embedded 2.4GHz radios

On a Raspberry Pi we will start by disabling the onboard WiFi chipset. For test purposes, to avoid any other interference with 2.4GHz activity also disable Bluetooth.

To do so, append the 2 lines below at the end of `/boot/config.txt`

``` 
dtoverlay=disable-wifi
dtoverlay=disable-bt
```

### Declaring the module in the device tree

Certain kernel version might allow the use of undeclared devices in th device tree `DT`

Often (if not everytime) this will prevent the wfx module from connecting to the WF200 : `wfx-sdio mmc2:0001:1: no compatible device found in DT`

Fortunately Silicon Labs provides a device tree source in their WFx200 [tools' repo](https://github.com/SiliconLabs/wfx-linux-tools/blob/SD5/overlays/wfx-sdio-overlay.dts)

To use it download it in your wfx workspace using `wget https://raw.githubusercontent.com/SiliconLabs/wfx-linux-tools/SD5/overlays/wfx-sdio-overlay.dts`

**THIS CANNOT BE USED AS IS**

When using the mainstream kernel module, *compatible* should be set to `compatible = "silabs,wf200";`

The wfx-linux-tools .dts file uses `compatible = "silabs,wfx-sdio";` which will prevent SDIO detection to occur

Once edited, we will use dtc (Device Tree Compiler) to compile this into a .dtbo (Device Tree Blob) file using `dtc -@ -I dts -O dtb -o wfx-sdio.dtbo wfx-sdio-overlay.dts`.

dtc is available from apt :`apt install device-tree-compiler`

This should create a new file named *wfx-sdio.dtbo*. Simply copy this into `/boot/overlays` as super user:

`cp wfx-sdio.dtbo /boot/overlays/`

Finally enable the overlay by adding `dtoverlay=wfx-sdio` to `/boot/config.txt` :

```
dtoverlay=wfx-sdio,sdio_overclock=25
```

*Note : Silicon Labs has also added a power sequence in the ovrelay source. It is recomended to use it as well*

*Note 2 : Silicon Labs' DTS will enable sdio on the RPi and disable the one used by the brcm-wifi*

*Note 3: The Silicon Labs Hat has a hardware limitation preventing it work faster than 25MHz, pass along sdio_overclock=25 to the dtoverlay config*

### Checking sdio on the Raspberry Pi hardware 

To check that SDIO is up and running, start by pushing device tree contents into a file named `device_tree.out` :

``` console
dtc --sort -I fs -O dts  /sys/firmware/devicetree/base > device_tree.out
```

Then look for `sdio_pins` into it `grep -A 5 sdio_pins device_tree.out` and check the pinout:

``` console
pi@raspberrypi:~ $ grep -A 5 sdio_pins device_tree.out
                sdio_pins = "/soc/gpio@7e200000/sdio_pins";
                smi = "/soc/smi@7e600000";
                soc = "/soc";
                sound = "/soc/sound";
                spi = "/soc/spi@7e204000";
                spi0 = "/soc/spi@7e204000";
--
                        sdio_pins {
                                brcm,function = <0x07>;
                                brcm,pins = <0x16 0x17 0x18 0x19 0x1a 0x1b>;
                                brcm,pull = <0x00 0x02 0x02 0x02 0x02 0x02>;
                                phandle = <0x97>;
                        };
```

Finally identify the mmc interface used for sdio by rebooting and issuing `dmesg | grep 'new high speed SDIO'` :

``` console 
pi@raspberrypi:~ $ dmesg | grep 'new high speed SDIO'
[    3.148442] mmc2: new high speed SDIO card at address 0001
```

Check that everything is fine by issuing `cat /sys/kernel/debug/mmc2/ios` :

```console 
pi@raspberrypi:~ $ sudo cat /sys/kernel/debug/mmc2/ios
clock:          50000000 Hz
actual clock:   25000000 Hz
vdd:            21 (3.3 ~ 3.4 V)
bus mode:       2 (push-pull)
chip select:    0 (don't care)
power mode:     2 (on)
bus width:      2 (4 bits)
timing spec:    2 (sd high-speed)
signal voltage: 0 (3.30 V)
driver type:    0 (driver type B)
```

### Downloading and setting up WFx200 SEC firmware 

WFx200 has no firmware and is required to be flashed by the wfx driver upon startup

By default the driver will look in `/lib/firmware/wfm_wf200.sec`

To download the latest version start by cloning the wfx-firmare repo :

```console 
cd ~/wfx_driver
git clone https://github.com/SiliconLabs/wfx-firmware.git
cd wfx-firmware
```

Once done copy the .sec file provided in the destination directory as super user:

```console 
cp wfm_wf200_C0.sec /lib/firmware/wfm_wf200.sec
```

### Generating and setting up the WFx200 PDS file

WFx200 also needs a PDS (Platform Data Set) file to be fed to the chipset

By default the driver will look in `/lib/firmware/wf200.pds`

To download the latest version start by cloning the wfx-firmare repo :

```console 
cd ~/wfx_driver
git clone https://github.com/SiliconLabs/wfx-firmware.git
```

Once done, you will also need the PDS compress Python script availablein the *wfx-linux-tools* repository :

```console 
git clone https://github.com/SiliconLabs/wfx-linux-tools
```

cd into wfx-firmware : `cd wfx-firmware`

There run `python3 ../wfx-linux-tools/pds_compress PDS/template.pds.in wf200.pds`

Once done copy the .pds file provided in the destination directory as super user:

```console 
cp wf200.pds /lib/firmware/wf200.pds
```

### Reboot and start using the WFx200

### Debugging sdio detection on the Raspberry Pi hardware 

* Error : `mmc2: error -110 whilst initialising SDIO card`
This usually is due to clock speed being too high. 

Silcion Labs shared hardware limitations related to their eval boards on [this KBA](https://community.silabs.com/s/article/linux-sdio-detection?language=en_US) sction "Max SDIO speed too high": 

*There is a SDIO speed limitation with BRD8022/BRD8023 due to the onboard switches. For this reason, we need to reduce the SDIO clock speed. 25 MHz is generally a good choice, given that choices are limited and vary depending on the platform.*

If enabling traces reflect the -110 error in `dmesg` then reduce clock speed by downclocking the bus in `/boot/config.txt`: 
```
dtoverlay=wfx-sdio,sdio_overclock=25
```

Another  way to debug Device Tree is to use /boot/config.txt as follows:
```
# Enable DRM VC4 V3D driver
#dtoverlay=vc4-kms-v3d
#max_framebuffers=2

dtdebug=1
```

And `vcdbg log msg`

