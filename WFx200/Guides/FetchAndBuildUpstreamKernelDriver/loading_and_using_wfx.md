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

## Using the kernel module to enable the WF200

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

By default the driver will look in `/lib/firmware/wfx/wfm_wf200.sec`

To download the latest version start by cloning the wfx-firmare repo :

```console
cd ~/wfx_driver
git clone https://github.com/SiliconLabs/wfx-firmware.git
cd wfx-firmware
```

Once done copy the .sec file provided in the destination directory as super user:

```console
cp wfm_wf200_C0.sec /lib/firmware/wfx/
```

```
wget https://github.com/SiliconLabs/wfx-firmware/raw/FW3.17.0/wfm_wf200_C0.sec
```

### Generating and setting up the WFx200 PDS file

WFx200 also needs a PDS (Platform Data Set) file to be fed to the chipset

By default the driver will look in `/lib/firmware/wfx/wf200.pds`

*** IMPORTANT ** When using mainsteam kernel, the PDS file format is different:

Mainstream didn't allow Silicon Labs to publish with their initial format. Silicon Labs  had to change to a tlv format (type/length/value), which needs to be generated from a new version of pds_compress, one that supports the tlv format.

Get the latest pds_compress script :

`wget https://raw.githubusercontent.com/SiliconLabs/wfx-linux-tools/master/pds_compress`

Add execute permissions to the script :

`sudo chmod +x pds_compress`

Get the PDS of your choice. I wil use the one dedicated to BRD8022 :

`wget https://raw.githubusercontent.com/SiliconLabs/wfx-pds/API3.0/BRD8022A_Rev_A06.pds.in`

Get the definitions.in that match your firmware version. In our case v3.17:

`wget https://raw.githubusercontent.com/SiliconLabs/wfx-firmware/FW3.17.0/PDS/definitions.in`

Modify the PDS file by commenting out 2 lines :

```
// FRONT_END_LOSS_CORRECTION_QDB
// RSSI_CORRECTION
```

And run the script :

`pds_compress -l BRD8022A_Rev_A06.pds.in wf200.pds`

Once done copy the .pds file provided in the destination directory as super user:

```console
cp wf200.pds /lib/firmware/wfx/wf200.pds
```

### Reboot and start using the WFx200

```
lab@raspberrypi:~ $ dmesg | grep wfx
[    6.836028] wfx: loading out-of-tree module taints kernel.
[    6.978024] wfx-sdio mmc3:0001:1: started firmware 3.17.0 "WF200_ASIC_WFM_(Jenkins)_FW3.17.0" (API: 3.12, keyset: C0, caps: 0x00000002)
[    7.000346] wfx-sdio mmc3:0001:1: MAC address 0: 00:0d:6f:73:93:69
[    7.000424] wfx-sdio mmc3:0001:1: MAC address 1: 00:0d:6f:73:93:6a
```

## Troubleshoot

### Debugging sdio detection on the Raspberry Pi hardware

- Error : `mmc2: error -110 whilst initialising SDIO card`
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

### Debugging .sec injection into wfx200

```
dmesg | grep wfx
[    6.651443] wfx: loading out-of-tree module taints kernel.
[    6.687967] wfx-sdio mmc2:0001:1: can't load wfx/wfm_wf200_C0.sec, falling back to wfx/wfm_wf200.sec
[    6.688083] wfx-sdio mmc2:0001:1: Direct firmware load for wfx/wfm_wf200.sec failed with error -2
[    6.688119] wfx-sdio mmc2:0001:1: can't load wfx/wfm_wf200.sec
[    6.694957] wfx-sdio: probe of mmc2:0001:1 failed with error -2
```

```
dmesg | grep wfx
[    6.640329] wfx: loading out-of-tree module taints kernel.
[    6.691847] wfx-sdio mmc2:0001:1: can't load wfx/wfm_wf200_C0.sec, falling back to wfx/wfm_wf200.sec
```

### Debugging .pds injection into wfx200

```
pi@raspberrypi:~ $ dmesg | grep wfx
[    6.809445] wfx: loading out-of-tree module taints kernel.
[    6.867018] wfx-sdio mmc2:0001:1: can't load wfx/wfm_wf200_C0.sec, falling back to wfx/wfm_wf200.sec
[    7.085869] wfx-sdio mmc2:0001:1: started firmware 3.17.0 "WF200_ASIC_WFM_(Jenkins)_FW3.17.0" (API: 3.12, keyset: C0, caps: 0x00000002)
[    7.086069] wfx-sdio mmc2:0001:1: Direct firmware load for wfx/wf200.pds failed with error -2
[    7.086118] wfx-sdio mmc2:0001:1: can't load antenna parameters (PDS file wfx/wf200.pds). The device may be unstable.
[    7.112949] wfx-sdio mmc2:0001:1: timeout while wake up chip
[    7.128974] wfx-sdio mmc2:0001:1: timeout while wake up chip
[    7.144951] wfx-sdio mmc2:0001:1: timeout while wake up chip
[    7.164905] wfx-sdio mmc2:0001:1: max wake-up retries reached
[    7.171020] wfx-sdio mmc2:0001:1: wfx_data_write: bus communication error: -84
[    8.133000] wfx-sdio mmc2:0001:1: chip is abnormally long to answer
[   11.237016] wfx-sdio mmc2:0001:1: chip did not answer
[   11.242318] wfx-sdio mmc2:0001:1: hardware request WRITE_MIB/GL_OPERATIONAL_POWER_MODE (0x06) on vif 2 returned error -110
[   11.254156] wfx-sdio mmc2:0001:1: MAC address 0: 00:0d:6f:73:93:69
[   11.254506] wfx-sdio mmc2:0001:1: MAC address 1: 00:0d:6f:73:93:6a
```

```
dmesg | grep wfx
[    6.767220] wfx: loading out-of-tree module taints kernel.
[    7.042675] wfx-sdio mmc2:0001:1: started firmware 3.17.0 "WF200_ASIC_WFM_(Jenkins)_FW3.17.0" (API: 3.12, keyset: C0, caps: 0x00000002)
[    7.296633] wfx-sdio mmc2:0001:1: PDS: malformed file (legacy format?)
[    7.304044] wfx-sdio: probe of mmc2:0001:1 failed with error -22
```

### WFx200 wakeup

```
pi@raspberrypi:~ $ dmesg | grep wfx
[    6.694119] wfx: loading out-of-tree module taints kernel.
[    7.011533] wfx-sdio mmc2:0001:1: started firmware 3.17.0 "WF200_ASIC_WFM_(Jenkins)_FW3.17.0" (API: 3.12, keyset: C0, caps: 0x00000002)
[    7.360657] wfx-sdio mmc2:0001:1: MAC address 0: 00:0d:6f:73:93:69
[    7.361673] wfx-sdio mmc2:0001:1: MAC address 1: 00:0d:6f:73:93:6a
[   13.213003] wfx-sdio mmc2:0001:1: timeout while wake up chip
```

Hint: to avoid the 'timeout while wake up chip', it is possible to comment (in the device tree) the wfx_wakeup segment. most Linux platforms won't really need to use device power save anyway (Saving the additional mW is generally not an issue on Linux platforms as it is on battery-powered IoT products).
