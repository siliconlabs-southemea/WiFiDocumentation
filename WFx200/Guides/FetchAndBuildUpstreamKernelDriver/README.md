---
sort: 1
---

# WFx200 - Fetching And Building Linux Kernel Upstream Driver locally

{% include list.liquid all=true %}

## Documentation ##

Silicon Labs states that from their [Vendor Repo]() the driver sources support only Linux Kernel < 5.17, therefore redirecting us towards the linux mainline repository

Unfortunately, it appears that depending on the distro you will be using, you might want to fetch sources from different origins. As a consequence, up to three build flows can be followed :

1. Build sources along the whole Kernel by including the WFX driver in .config
2. Build only the kernel module 
    - *On distros where WFX is not based out of linux upstream (i.e. RaspiOS)*
    - On regular distributions

This guide will be covering RaspiOS build of the kernel module only, on kernel versions > 5.17

## Disclaimer ##

The Gecko SDK suite supports development with Silicon Labs IoT SoC and module devices. Unless otherwise specified in the specific directory, all examples are considered to be EXPERIMENTAL QUALITY which implies that the code provided in the repos has not been formally tested and is provided as-is.  It is not suitable for production environments.  In addition, this code will not be maintained and there may be no bug maintenance planned for these resources. Silicon Labs may update projects from time to time.
