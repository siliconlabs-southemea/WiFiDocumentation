---
sort: 2
---

# Building and installing WFx kernel module from upstream branch

At this point you should have :
 - Downloaded Kernel headers matching your running platform
 - Downloaded Kernel sources matching your running platform

## Building the wfx Kernel Object 

Building the entire kernel takes too much time, so the below focuses on building the WFx module only

Once you have downloaded the sources navigate to the WFX driver :
``` console
cd linux-stable/drivers/net/wireless/silabs/wfx/
```

Once there simply build the module by calling `make -C /lib/modules/$(uname -r)/build/ CONFIG_WFX=m M=$(pwd) modules` :

If `$(uname -r)` does not return the proper header's directory, adjust to the correct value in the command instead

``` console 
pi@raspberrypi:~/wfx_driver/linux-stable/drivers/net/wireless/silabs/wfx $ make -C /lib/modules/$(uname -r)/build/ CONFIG_WFX=m M=$(pwd) modules
make: Entering directory '/usr/src/linux-headers-6.1.21-v8+'
  CC [M]  /home/pi/wfx_driver/linux-stable/drivers/net/wireless/silabs/wfx/bh.o
  CC [M]  /home/pi/wfx_driver/linux-stable/drivers/net/wireless/silabs/wfx/hwio.o
  CC [M]  /home/pi/wfx_driver/linux-stable/drivers/net/wireless/silabs/wfx/fwio.o
  CC [M]  /home/pi/wfx_driver/linux-stable/drivers/net/wireless/silabs/wfx/hif_tx_mib.o
  CC [M]  /home/pi/wfx_driver/linux-stable/drivers/net/wireless/silabs/wfx/hif_tx.o
  CC [M]  /home/pi/wfx_driver/linux-stable/drivers/net/wireless/silabs/wfx/hif_rx.o
  CC [M]  /home/pi/wfx_driver/linux-stable/drivers/net/wireless/silabs/wfx/queue.o
  CC [M]  /home/pi/wfx_driver/linux-stable/drivers/net/wireless/silabs/wfx/data_tx.o
  CC [M]  /home/pi/wfx_driver/linux-stable/drivers/net/wireless/silabs/wfx/data_rx.o
  CC [M]  /home/pi/wfx_driver/linux-stable/drivers/net/wireless/silabs/wfx/scan.o
  CC [M]  /home/pi/wfx_driver/linux-stable/drivers/net/wireless/silabs/wfx/sta.o
  CC [M]  /home/pi/wfx_driver/linux-stable/drivers/net/wireless/silabs/wfx/key.o
  CC [M]  /home/pi/wfx_driver/linux-stable/drivers/net/wireless/silabs/wfx/main.o
  CC [M]  /home/pi/wfx_driver/linux-stable/drivers/net/wireless/silabs/wfx/debug.o
  CC [M]  /home/pi/wfx_driver/linux-stable/drivers/net/wireless/silabs/wfx/bus_spi.o
  CC [M]  /home/pi/wfx_driver/linux-stable/drivers/net/wireless/silabs/wfx/bus_sdio.o
  LD [M]  /home/pi/wfx_driver/linux-stable/drivers/net/wireless/silabs/wfx/wfx.o
  MODPOST /home/pi/wfx_driver/linux-stable/drivers/net/wireless/silabs/wfx/Module.symvers
  CC [M]  /home/pi/wfx_driver/linux-stable/drivers/net/wireless/silabs/wfx/wfx.mod.o
  LD [M]  /home/pi/wfx_driver/linux-stable/drivers/net/wireless/silabs/wfx/wfx.ko
make: Leaving directory '/usr/src/linux-headers-6.1.21-v8+'
```

As seen in the console trace, the kernel module is built `/home/pi/wfx_driver/linux-stable/drivers/net/wireless/silabs/wfx/wfx.ko`

## Installing the wfx Kernel Object 

Still from the same directory now we will install the module on the running target by calling `make -C /lib/modules/$(uname -r)/build/ CONFIG_WFX=m M=$(pwd) modules_install` as superuser

Again, if `$(uname -r)` does not return the proper header's directory, adjust to the correct value in the command instead


``` console 
pi@raspberrypi:~/wfx_driver/linux-stable/drivers/net/wireless/silabs/wfx $ sudo make -C /lib/modules/$(uname -r)/build/ CONFIG_WFX=m M=$(pwd) modules_install
make: Entering directory '/usr/src/linux-headers-6.1.21-v8+'
  INSTALL /lib/modules/6.1.21-v8+/extra/wfx.ko
  XZ      /lib/modules/6.1.21-v8+/extra/wfx.ko.xz
  DEPMOD  /lib/modules/6.1.21-v8+
Warning: modules_install: missing 'System.map' file. Skipping depmod.
make: Leaving directory '/usr/src/linux-headers-6.1.21-v8+'
``` 

Finally you might need to run `depmod -a` as superuser manually afterwards if the warning shows up:

``` console
pi@raspberrypi:~/wfx_driver/linux-stable/drivers/net/wireless/silabs/wfx $ sudo depmod -A
```

