---
sort: 3
---

# SiWG917 software control for low power:

## Introduction:

As described in previous chapter, SiWG917 SoC consist of two cores: 
- ThreadArch (TA) running the RF stacks
- Cortex M4 running the SoC application code

Therefore we need to control both cores and be aware of their states to be able to achieve the lowest possible power in each core and for the full part.

## ThreadArch stack core:

ThreadArch core cannot run application code. Therefore the way to control its state is through its initialization and/or control using stacks APIs.

### Initialization

API function call to initialize radio stacks is : 

https://docs.silabs.com/wiseconnect/3.1.0/wiseconnect-api-reference-guide-nwk-mgmt/net-interface-functions#sl-net-init
```c
sl_status_t sl_net_init ( 
    sl_net_interface_t interface, 
    const void *configuration, 
    void *context, 
    sl_net_event_handler_t event_handler)
```

* the **"interface"** setting defines what radio stack you want to use out of WIFI (AP/STA) or BLE. in our case, you can chose any setting from the SIG917 available ones on https://docs.silabs.com/wiseconnect/3.1.0/wiseconnect-api-reference-guide-nwk-mgmt/sl-net-constants#sl-net-interface-t

* second parameter is **"configuration"**. It will be structure like sl_wifi_device_configuration_t (https://docs.silabs.com/wiseconnect/3.1.0/wiseconnect-api-reference-guide-si91x-driver/sl-wifi-device-configuration-t).
See code examples.

* third one **"context"** can be set to NULL most of the time but depends on the "interface" setting.

* last one **"event_handler"** can also be set to NULL in this example as we will not use the radio stack.

### Performance profile

The next API function call is the one important for the power level:
https://docs.silabs.com/wiseconnect/3.1.0/wiseconnect-api-reference-guide-wi-fi/wifi-power-api#sl-wifi-set-performance-profile

```c
sl_status_t sl_wifi_set_performance_profile (const sl_wifi_performance_profile_t *profile)
```

* the "profile" structure is available on: https://docs.silabs.com/wiseconnect/3.0.13/wiseconnect-api-reference-guide-si91x-driver/sl-wifi-performance-profile-t

* in this structure, the **"profile"** entry will dictate power consumption. 

* Options are listed on https://docs.silabs.com/wiseconnect/3.0.13/wiseconnect-api-reference-guide-si91x-driver/sl-si91-x-constants#sl-performance-profile-t

    * **"STANDBY_POWER_SAVE"** : Power save mode when module is not associated with either AP or station, ram is not retained in this mode

    * **"STANDBY_POWER_SAVE_WITH_RAM_RETENTION"** : Power save mode when module is not associated with either AP or station, ram is retained in this mode. 

lowest power figures are obtained with **"STANDBY_POWER_SAVE"** as RAM is not retained.


## M4 application core:

M4 core can run application code. Therefore the way to control its state is by directly setting its state mode using the available sleep control.

Those are the functions to be used (from "Si91x SoC" component):

```c 
void sl_si91x_configure_ram_retention(uint32_t rams_in_use, uint32_t rams_retention_during_sleep)
```
This function defines strategy for RAM used in active mode as well as the retention in sleep mode.

* **"rams_in_use"** defines the RAMs to be powered functionally (the rest of the RAM banks are off)
    * **WISEMCU_0KB_RAM_IN_USE** : no RAM used
    * **WISEMCU_16KB_RAM_IN_USE** : Only 16KB  RAM used
    * **WISEMCU_64KB_RAM_IN_USE** : Only 64KB  RAM used
    * **WISEMCU_128KB_RAM_IN_USE** : Only 128KB  RAM used
    * **WISEMCU_192KB_RAM_IN_USE** : Only 192KB  RAM used
    * **WISEMCU_256KB_RAM_IN_USE** : Only 256KB  RAM used
    * **WISEMCU_320KB_RAM_IN_USE** : all 320KB  RAM used

* **"rams_retention_during_sleep"** Configure which RAM blocks are retained during sleep
    * **WISEMCU_RETAIN_DEFAULT_RAM_DURING_SLEEP** : 320kB for SiWG917
    * **WISEMCU_RETAIN_16K_RAM_DURING_SLEEP**     : Retain 16KB M4-ULP RAM
    * **WISEMCU_RETAIN_128K_RAM_DURING_SLEEP**    : Retain 16KB M4-ULP RAM and 112KB M4-ULP RAM
    * **WISEMCU_RETAIN_192K_RAM_DURING_SLEEP**    : Retain 16KB M4-ULP RAM and 112KB M4-ULP RAM and 64KB M4SS RAM
    * **WISEMCU_RETAIN_384K_RAM_DURING_SLEEP**    : Retain 16KB M4-ULP RAM and 112KB M4-ULP RAM and 64KB M4SS RAM and TASS 192KB RAM
    * **WISEMCU_RETAIN_M4SS_RAM_DURING_SLEEP**   : Retain Only 64KB M4SS RAM
    * **WISEMCU_RETAIN_ULPSS_RAM_DURING_SLEEP**  : Retain Only 16KB ULPSS RAM
    * **WISEMCU_RETAIN_TASS_RAM_DURING_SLEEP**    : Retain Only 192KB TASS RAM
    * **WISEMCU_RETAIN_M4ULP_RAM_DURING_SLEEP**  : Retain Only 112KB M4-ULP RAM



```c 
void sl_si91x_trigger_sleep(SLEEP_TYPE_T sleepType,
                            uint8_t lf_clk_mode,
                            uint32_t stack_address,
                            uint32_t jump_cb_address,
                            uint32_t vector_offset,
                            uint32_t mode)
```
This function triggers M4 core to sleep.

* first parameter is **"sleepType"**, this is the one of most interest. Settings are:
    * **SLEEP_WITH_RETENTION** : RAM is retained as per above function settings 
    * **SLEEP_WITHOUT_RETENTION** : RAM is not retained

* second parameter is **"lf_clk_mode"**. This parameter is used to switch the processor clock from high frequency clock to low-frequency clock. This is used in some critical power save cases.
    * **"0"**: disable LF mode, normal mode used by default
    * **"1"** : LF 32K RC, Processor clock is configured to low-frequency RC clock.
    * **"2"** : LF 32K XTAL, Processor clock is configured to low-frequency XTAL clock.

* third parameter is **"stack_address"**, it is the Stack pointer address to be used by bootloader.
* forth parameter is **"jump_cb_address"**,it controls block memory address or function address to be branched up on Wake-up
* fifth parameter is **"vector_offset"**, it is the IVT offset to be programmed by boot-loader up on Wake-up.
* sixth and last parameter is **"mode"** set with:
    * **RSI_WAKEUP_FROM_FLASH_MODE**  : Wakes from flash with retention. Upon wake up, control jumps to wake up handler in flash. In this mode, ULPSS RAMs are used to store the stack pointer and Wake-up handler address.
    * **RSI_WAKEUP_WITH_OUT_RETENTION** : Without retention sleep common for both FLASH/RAM based execution. In this mode, ULPSS RAMs are used to store the stack pointer and control block address. if stack_addr and jump_cb_addr are not valid, then 0x2404_0C00 and 0x2404_0000 are usedfor stack and control block address respectively.
    * **RSI_WAKEUP_WITH_RETENTION **: With retention branches to wake up handler in RAM. In this mode, ULPSS RAMs are used to store the wake up handler address.
    * **RSI_WAKEUP_WITH_RETENTION_WO_ULPSS_RAM** : In this mode, ULPSS RAMs are not used by boot-loader, instead it uses the NPSS battery flip flops.
    * **RSI_WAKEUP_WO_RETENTION_WO_ULPSS_RAM**  : In this mode, ULPSS RAMs are not used by boot-loader, instead it uses the NPSS battery flip flops to store the stack and derives the control block address by adding 0XC00 to the stack address stored in battery flops.
