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

TODO