---
sort: 4
---

# Code exemple and measurements results:

## Code example:

you can download here the <a href="./images/app.c" download>app.c</a> file.

create a project for your BRD4338A board in Simplicity Studio v5, choose for example "Wi-Fi - Powersave Deep Sleep (SOC)" as it already integrate all the components needed for the above application.

then replace the project app.c file with the one from this page.

line 143 defines ThreadArch performance profile to STANDBY_POWER_SAVE. As just the sl_net_init() is done, and no wifi AP/client or ble configuration is done, ThreadArch will automaticaly switch to the power save mode reaching its lowest current point.

```c
sl_wifi_performance_profile_t performance_profile = { .profile = STANDBY_POWER_SAVE };

sl_wifi_set_performance_profile(&performance_profile);
```
Then setting ram usage and retention sizes profiles in line 160 and enabling retention or not in line 163 will define directly current in sleep mode.

```c
sl_si91x_configure_ram_retention(WISEMCU_128KB_RAM_IN_USE, WISEMCU_RETAIN_DEFAULT_RAM_DURING_SLEEP);

/* Trigger M4 Sleep*/
sl_si91x_trigger_sleep(SLEEP_WITHOUT_RETENTION,
                          DISABLE_LF_MODE,
                          WKP_RAM_USAGE_LOCATION,
                          (uint32_t)RSI_PS_RestoreCpuContext,
                          IVT_OFFSET_ADDR,
                          RSI_WAKEUP_FROM_FLASH_MODE);
```
## Measurements:

Measurements are done using Simplicity Studio v5 Energy Profiler.  

There is an exemple capture as per above and per default setting of app.c provided in that page.

<img src="./images/capture no retention.png" alt="SiWG917 Block Diagram" width="1024" class="center">    

you can see on the left current from the reset state, then you see going to the right SiWG917 boot, ThreadArch configuratution, then finally full Sleep mode state.

Here are the results for this last section current measurement @3.3V @25C .

* ThreadArch in STANDBY_POWER_SAVE and M4 in SLEEP_WITHOUT_RETENTION : **2.82uA**
    * other settings are unused in that mode.

* ThreadArch in STANDBY_POWER_SAVE and M4 in SLEEP_WITH_RETENTION : **4.05uA**
    * WISEMCU_128KB_RAM_IN_USE
    * WISEMCU_RETAIN_ULPSS_RAM_DURING_SLEEP

* ThreadArch in STANDBY_POWER_SAVE and M4 in SLEEP_WITH_RETENTION : **16.11uA**
    * WISEMCU_128KB_RAM_IN_USE
    * WISEMCU_RETAIN_128K_RAM_DURING_SLEEP

* ThreadArch in STANDBY_POWER_SAVE and M4 in SLEEP_WITH_RETENTION : **15.49uA**
    * WISEMCU_128KB_RAM_IN_USE
    * WISEMCU_RETAIN_M4ULP_RAM_DURING_SLEEP
