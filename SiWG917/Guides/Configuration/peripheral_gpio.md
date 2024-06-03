---
sort: 2
---

# SiWG917 peripheral MUX to GPIOs:

## 1- Block Diagram:

<img src="./images/917 mux.png" alt="SiWG917 peripheral and gpios block diagram" width="1024" class="center">    
 

SiWG917 SoC consist of two cores: 
- ThreadArch (TA) running the RF stacks
- Cortex M4 running the SoC application code

M4 Peripherals is divided in two sets:  
- high speed/ high performance ones
- Ultra Low Power (ULP) optimized ones  

 All this is fundamental for the understanding of what can be done and when.
Indeed, **both cores have somehow an independant life and setting a power state has also an impact on the availability of some peripherals**. 

## 2- RCP/NCP (N-link/Wiseconnect) modes:

SiWG917 in RCP/NCP modes do not expose M4 domain.  

ThreadArch core peripherals are exposed to the GPIOs through high speed GPIO MUX mode 15.

Therefore for RCP/NCP references, peripheral names are given through this table:

**Table 0:**


|Pad Selection|Mode 0|Mode 1|Mode 2|Mode 3|Mode 4|Mode 5|Mode 6|Mode 7|
| :- | :- | :- | :- | :- | :- | :- | :- | :- |
|1|GPIO\_6|UART2\_RX|I2S\_DOUT|RF\_SPI\_CLK|TCK|NWP\_CLK\_OUT|QSPI\_D0|ADC\_CLOCK\_STRM1|
|2|GPIO\_7|UART2\_TX|I2S\_CLK|RF\_SPI\_CSB|TDI|MODEM\_GPIO3\_BAND1|QSPI\_CSN0|DAC\_CLOCK\_STRM1|
|3|GPIO\_8|UART1\_RX|LED\_0|RF\_SPI\_D0|RF\_TEST\_2|MODEM\_GPIO4\_BAND1|QSPI\_CLK|MODEM\_PLL160\_CLOCK|
|4|GPIO\_9|UART1\_TX|LED\_1|RF\_SPI\_D1|AFE\_TEST\_2|MODEM\_GPIO5\_BAND1|QSPI\_D1|MODEM\_PLL44\_CLOCK|
|5|GPIO\_10|UART2\_CTS|RF\_TEST\_1|RF\_SPI\_D2/RXON|TMS|I2S\_DIN|QSPI\_D2|MODEM\_PLL\_SOC1\_CLOCK|
|6|GPIO\_11|UART2\_RTS|AFE\_TEST\_1|RF\_SPI\_D3/TXON|TDO|I2S\_WS|QSPI\_D3|MODEM\_PLL\_SOC2\_CLOCK|
|7|GPIO\_12|UART2\_RX|QSPI\_CSN1|I2C\_SCL|UART1\_RTS|NWP\_CLK\_OUT|||
|8|GPIO\_15|UART2\_TX|SDMEM\_PRESENT|UART1\_CTS|I2C\_SDA|NWP\_CLK\_OUT|||
|10|GPIO\_46|QSPI\_CLK|UART1\_RS485\_EN|TCK|I2C\_SCL|ANT\_SEL\_A[0]|||
|11|GPIO\_47|QSPI\_D0|UART1\_RS485\_RE|TDI|I2C\_SDA|ANT\_SEL\_B[0]|||
|12|GPIO\_48|QSPI\_D1|UART1\_RS485\_DE|TDO|UART2\_RX|ANT\_SEL\_A[1]|||
|13|GPIO\_49|QSPI\_CSN0|UART2\_RS485\_EN|TMS|MODEM\_GPIO0\_BAND1|ANT\_SEL\_B[1]|||
|14|GPIO\_50|QSPI\_D2|UART2\_RS485\_RE|TX\_PA\_ON|MODEM\_GPIO1\_BAND1|ANT\_SEL\_A[1]|||
|15|GPIO\_51|QSPI\_D3|UART2\_RS485\_DE|UART2\_TX|MODEM\_GPIO2\_BAND1|ANT\_SEL\_B[1]|||

Those are not accessible to the M4 code and are just given for your understanding of peripheral exposed in RCP/NCP applications.

## 2- SOC mode (user code running on M4 core):

SiWG917 in SOC mode exposes M4 domain.

Available peripherals are separated into 2 groups:
- active PS4/3 modes when high speed clocks are running
- sleep modes when only the ultra low power clocks are running

Exclusively in active mode and only for the digital peripherals, there are some MUX options to uexpose PS4/3 M4 periperals on the ULP GPIOs but also ULP peripherals on PS4/3 GPIOs.

Those are visible from the block diagram through the arrows indicating in which MUX mode they are available.

You can see them in Table 1.1/1.2 and Table 2 below.

Note that Table 1.2 MUX options can only be used through the ULP GPIOs on the SiW917.

**Table 1.1:**


| **PIN** | **PAD NUMBER** | **GPIO** | **GPIO Mode= 0** | **GPIO Mode= 1** | **GPIO Mode= 2** | **GPIO Mode= 3** | **GPIO Mode= 4** | **GPIO Mode= 5** | **GPIO Mode= 6** | **GPIO Mode= 7** | **GPIO Mode= 8** | **GPIO Mode= 9** | **GPIO Mode= 10** | **GPIO Mode= 11** | **GPIO Mode= 12** | **GPIO Mode*= 13** | **GPIO Mode*= 14** | **GPIO Mode*= 15** |
| ------- | -------------- | ---------------------- | --------------------- | --------------------- | --------------------- | --------------------- | --------------------- | --------------------- | --------------------- | --------------------- | --------------------- | --------------------- | -------------------------- | -------------------------- | -------------------------- | -------------------------- | -------------------------- | -------------------------- |
| 6 | 1 | GPIO_6 | GPIO_6 | SIO_0 | USART1_CTS | SSI_MST_DATA2 | I2C1_SDA | I2C2_SCL | UART2_RX | I2S_2CH_DIN_1 | PMU_TEST_1 | ULPPERH_ON_SOC_GPIO_0 | PWM_1L | M4SS_QSPI_D0 | GSPI_MST1_MOS1 | M4SS_TRACE_CLKIN |  | NWP_GPIO_6 |
| 7 | 2 | GPIO_7 | GPIO_7 | SIO_1 | USART1_DTR | SSI_MST_DATA3 | I2C1_SCL | I2C2_SDA | UART2_TX | I2S_2CH_DOUT_1 | PMU_TEST_2 | ULPPERH_ON_SOC_GPIO_1 | PWM_1H | M4SS_QSPI_CSN0 | M4SS_QSPI_CSN1 | M4SS_TRACE_CLK |  |  |
| 8 | 3 | GPIO_8 | GPIO_8 | SIO_2 | USART1_CLK | SSI_MST_CLK | GSPI_MST1_CLK | QEI_IDX | UART2_RS485_RE | I2S_2CH_CLK | SSI_SLV_CLK | ULPPERH_ON_SOC_GPIO_2 | PWM_2L | M4SS_QSPI_CLK | SCT_OUT_2 | M4SS_TRACE_D0 |  | NWP_GPIO_8 |
| 9 | 4 | GPIO_9 | GPIO_9 | SIO_3 | USART1_RTS | SSI_MST_CS0 | GSPI_MST1_CS0 | QEI_PHA | UART2_RS485_DE | I2S_2CH_WS | SSI_SLV_CS |  | PWM_2H | M4SS_QSPI_D1 | SCT_OUT_3 | M4SS_TRACE_D1 |  | NWP_GPIO_9 |
| 10 | 5 | GPIO_10 | GPIO_10 | SIO_4 | USART1_RX | SSI_MST_CS1 | GSPI_MST1_CS1 | QEI_PHB | UART2_RTS | I2S_2CH_DIN_0 | SSI_SLV_MOSI | ULPPERH_ON_SOC_GPIO_4 | PWM_3L | M4SS_QSPI_D2 | SSI_MST_DATA1 | M4SS_TRACE_D2 |  | NWP_GPIO_10 |
| 11 | 6 | GPIO_11 | GPIO_11 | SIO_5 | USART1_DSR | SSI_MST_DATA0 | GSPI_MST1_MISO | QEI_DIR | UART2_CTS | I2S_2CH_DOUT_0 | SSI_SLV_MISO | ULPPERH_ON_SOC_GPIO_5 | PWM_3H | M4SS_QSPI_D3 | MCU_CLK_OUT | M4SS_TRACE_D3 |  | NWP_GPIO_11 |
| 12 | 7 | GPIO_12 | GPIO_12 |  | USART1_DCD | SSI_MST_DATA1 | GSPI_MST1_MOSI |  | UART2_RS485_EN |  | MCU_CLK_OUT | ULPPERH_ON_SOC_GPIO_6 | PWM_4L |  |  |  |  | NWP_GPIO_12 |
| 15 | 8 | GPIO_15 | GPIO_15 | SIO_7 | UART2_TX | SSI_MST_CS2 | GSPI_MST1_CS2 |  | M4SS_TRACE_CLKIN |  | MCU_CLK_OUT | ULPPERH_ON_SOC_GPIO_7 | PWM_4H |  |  |  |  | NWP_GPIO_15 |
| 25 | No need to select the Pad | GPIO_25 | GPIO_25 | SIO_0 | USART1_CLK | SSI_MST_CLK | GSPI_MST1_CLK | QEI_IDX |  | I2S_2CH_CLK | SSI_SLV_CS | SCT_IN_0 | PWM_FAULTA | ULPPERH_ON_SOC_GPIO_6 | SOC_PLL_CLOCK | USART1_IR_RX | TopGPIO_0 |  |
| 26 | No need to select the Pad | GPIO_26 | GPIO_26 | SIO_1 | USART1_CTS | SSI_MST_DATA0 | GSPI_MST1_MISO | QEI_PHA | UART2_RS485_EN | I2S_2CH_WS | SSI_SLV_CLK | SCT_IN_1 | PWM_FAULTB | ULPPERH_ON_SOC_GPIO_7 | INTERFACE_PLL_CLOCK | USART1_IR_TX | TopGPIO_1 |  |
| 27 | No need to select the Pad | GPIO_27 | GPIO_27 | SIO_2 | USART1_RI | SSI_MST_DATA1 | GSPI_MST1_MOSI | QEI_PHB | UART2_RTS | I2S_2CH_DIN_0 | SSI_SLV_MOSI | SCT_IN_2 | PWM_TMR_EXT_TRIG_1 | ULPPERH_ON_SOC_GPIO_8 | I2S_PLL_CLOCK | USART1_RS485_EN | TopGPIO_2 |  |
| 28 | No need to select the Pad | GPIO_28 | GPIO_28 | SIO_3 | USART1_RTS | SSI_MST_CS0 | GSPI_MST1_CS0 | QEI_DIR | UART2_RTS | I2S_2CH_DOUT_0 | SSI_SLV_MISO | SCT_IN_3 | PWM_TMR_EXT_TRIG_2 | ULPPERH_ON_SOC_GPIO_9 | XTAL_ON_IN | USART1_RS485_RE | TopGPIO_3 |  |
| 29 | No need to select the Pad | GPIO_29 | GPIO_29 | SIO_4 | USART1_RX | SSI_MST_DATA2 | GSPI_MST1_CS1 | I2C2_SCL | UART2_RX | I2S_2CH_DIN_1 | PMU_TEST_1 | SCT_OUT_0 | PWM_TMR_EXT_TRIG_3 | ULPPERH_ON_SOC_GPIO_10 | USART1_DCD | USART1_RS485_DE | TopGPIO_4 |  |
| 30 | No need to select the Pad | GPIO_30 | GPIO_30 | SIO_5 | USART1_TX | SSI_MST_DATA3 | GSPI_MST1_CS2 | I2C2_SDA | UART2_TX | I2S_2CH_DOUT_1 | PMU_TEST_2 | SCT_OUT_1 | PWM_TMR_EXT_TRIG_4 | ULPPERH_ON_SOC_GPIO_11 | PMU_TEST_1 | PMU_TEST_2 | TopGPIO_5 |  |
| 31 | 9 | GPIO_31 | GPIO_31 |  |  |  |  |  |  |  |  |  |  | I2C1_SDA | UART2_RTS | QEI_IDX |  |  |
| 32 | 9 | GPIO_32 | GPIO_32 |  |  |  |  |  |  |  |  |  |  | I2C1_SCL | UART2_CTS | QEI_PHA |  |  |
| 33 | 9 | GPIO_33 | GPIO_33 |  |  |  |  |  |  |  |  |  |  | I2C2_SCL | UART2_RX | QEI_PHB |  |  |
| 34 | 9 | GPIO_34 | GPIO_34 |  |  |  |  |  |  |  |  |  |  | I2C2_SDA | UART2_TX | QEI_DIR |  |  |
| 46 | 10 | GPIO_46 | GPIO_46 | M4SS_QSPI_CLK | USART1_RI | QEI_IDX | GSPI_MST1_CLK |  | M4SS_TRACE_CLKIN | I2S_2CH_CLK | SSI_SLV_CS | ULPPERH_ON_SOC_GPIO_8 | SOC_PLL_CLOCK | M4SS_PSRAM_CLK |  |  |  | NWP_GPIO_46 |
| 47 | 11 | GPIO_47 | GPIO_47 | M4SS_QSPI_D0 | USART1_IR_RX | QEI_PHA | GSPI_MST1_MISO |  | M4SS_TRACE_CLK | I2S_2CH_WS | SSI_SLV_CLK | ULPPERH_ON_SOC_GPIO_9 | INTERFACE_PLL_CLOCK | M4SS_PSRAM_D0 |  |  |  | NWP_GPIO_47 |
| 48 | 12 | GPIO_48 | GPIO_48 | M4SS_QSPI_D1 | USART1_IR_TX | QEI_PHB | GSPI_MST1_MOSI |  | M4SS_TRACE_D0 | I2S_2CH_DIN_0 | SSI_SLV_MOSI | ULPPERH_ON_SOC_GPIO_10 | I2S_PLL_CLOCK | M4SS_PSRAM_D1 |  |  |  | NWP_GPIO_48 |
| 49 | 13 | GPIO_49 | GPIO_49 | M4SS_QSPI_CSN0 | USART1_RS485_EN | QEI_DIR | GSPI_MST1_CS0 |  | M4SS_TRACE_D1 | I2S_2CH_DOUT_0 | SSI_SLV_MISO | ULPPERH_ON_SOC_GPIO_11 | MCU_QSPI_CSN0 | M4SS_PSRAM_CSN0 |  |  |  | NWP_GPIO_49 |
| 50 | 14 | GPIO_50 | GPIO_50 | M4SS_QSPI_D2 | USART1_RS485_RE | SSI_MST_CS2 | GSPI_MST1_CS1 | I2C2_SCL | M4SS_TRACE_D2 | I2S_2CH_DIN_1 | PWM_TMR_EXT_TRIG_4 | UART2_RTS | MEMS_REF_CLOCK | M4SS_PSRAM_D2 |  |  |  | NWP_GPIO_50 |
| 51 | 15 | GPIO_51 | GPIO_51 | M4SS_QSPI_D3 | USART1_RS485_DE | SSI_MST_CS3 | GSPI_MST1_CS2 | I2C2_SDA | M4SS_TRACE_D3 | I2S_2CH_DOUT_1 | PWM_TMR_EXT_TRIG_1 | UART2_CTS | PLL_TESTMODE_SIG | M4SS_PSRAM_D3 |  |  |  | NWP_GPIO_51 |
| 52 | 16 | GPIO_52 | GPIO_52 |  | USART1_CLK | SSI_MST_CLK | GSPI_MST1_CLK | QEI_IDX | M4SS_TRACE_CLKIN | I2S_2CH_CLK | SSI_SLV_CLK | M4SS_QSPI_CLK | SOC_PLL_CLOCK |  | M4SS_PSRAM_CLK |  |  |  |
| 53 | 17 | GPIO_53 | GPIO_53 | M4SS_QSPI_CSN1 | USART1_RTS | SSI_MST_CS0 | GSPI_MST1_CS0 | QEI_PHA | M4SS_TRACE_CLK | I2S_2CH_WS | SSI_SLV_CS | M4SS_QSPI_D0 | INTERFACE_PLL_CLOCK | M4SS_PSRAM_CSN1 | M4SS_PSRAM_D0 |  |  |  |
| 54 | 18 | GPIO_54 | GPIO_54 | M4SS_QSPI_D4 | USART1_TX | SSI_MST_DATA2 | GSPI_MST1_CS1 | I2C2_SCL | M4SS_TRACE_D0 | I2S_2CH_DIN_1 | PWM_TMR_EXT_TRIG_2 | M4SS_QSPI_D1 | I2S_PLL_CLOCK | M4SS_PSRAM_D4 | M4SS_PSRAM_D1 |  |  |  |
| 55 | 19 | GPIO_55 | GPIO_55 | M4SS_QSPI_D5 | USART1_RX | SSI_MST_DATA3 | GSPI_MST1_CS2 | I2C2_SDA | M4SS_TRACE_D1 | I2S_2CH_DOUT_1 | PWM_TMR_EXT_TRIG_3 | M4SS_QSPI_CSN0 |  | M4SS_PSRAM_D5 | M4SS_PSRAM_CSN0 |  |  |  |
| 56 | 20 | GPIO_56 | GPIO_56 | M4SS_QSPI_D6 | USART1_CTS | SSI_MST_DATA0 | GSPI_MST1_MISO | QEI_PHB | M4SS_TRACE_D2 | I2S_2CH_DIN_0 | SSI_SLV_MOSI | M4SS_QSPI_D2 | MEMS_REF_CLOCK | M4SS_PSRAM_D6 | M4SS_PSRAM_D2 |  |  |  |
| 57 | 21 | GPIO_57 | GPIO_57 | M4SS_QSPI_D7 | USART1_DSR | SSI_MST_DATA1 | GSPI_MST1_MOSI | QEI_DIR | M4SS_TRACE_D3 | I2S_2CH_DOUT_0 | SSI_SLV_MISO | M4SS_QSPI_D3 | XTAL_ON_IN | M4SS_PSRAM_D7 | M4SS_PSRAM_D3 |  |  |  |


**Table 1.2:**


| **PIN** | **PAD NUMBER** | **GPIO** | **GPIO Mode= 0** | **GPIO Mode= 1** | **GPIO Mode= 2** | **GPIO Mode= 3** | **GPIO Mode= 4** | **GPIO Mode= 5** | **GPIO Mode= 6** | **GPIO Mode= 7** | **GPIO Mode= 8** | **GPIO Mode= 9** | **GPIO Mode= 10** | **GPIO Mode= 11** | **GPIO Mode= 12** | **GPIO Mode*= 13** |
| ------- | -------------- | ---------------------- | --------------------- | --------------------- | --------------------- | --------------------- | --------------------- | --------------------- | --------------------- | --------------------- | --------------------- | --------------------- | -------------------------- | -------------------------- | -------------------------- | -------------------------- |
| 64      | 22             | SOCPERH_ON_ULP_GPIO_0  | GPIO_64               | SIO_0                 | USART1_CLK            | QEI_IDX               | I2C1_SDA              | I2C2_SCL              | UART2_RS485_EN        | SCT_IN_0              | PWM_1L                | UART2_RTS             |                            | USART1_IR_RX               | PWM_1L                     | PMU_TEST_1                 |
| 65      | 23             | SOCPERH_ON_ULP_GPIO_1  | GPIO_65               | SIO_1                 | USART1_RX             | QEI_PHA               | I2C1_SCL              | I2C2_SDA              | UART2_RS485_RE        | SCT_IN_1              | PWM_1H                | UART2_CTS             |                            | USART1_IR_TX               | PWM_1H                     | PMU_TEST_2                 |
| 66      | 24             | SOCPERH_ON_ULP_GPIO_2  | GPIO_66               | SIO_2                 |                       | QEI_PHB               | I2C1_SCL              | I2C2_SCL              | UART2_RS485_DE        | SCT_IN_2              | PWM_2L                | UART2_RX              | PMU_TEST_1                 |                            |                            |                            |
| 67      | 25             | SOCPERH_ON_ULP_GPIO_3  | GPIO_67               | SIO_3                 |                       | QEI_DIR               | I2C1_SDA              | I2C2_SDA              |                       | SCT_IN_3              | PWM_2H                | UART2_TX              | PMU_TEST_2                 |                            |                            |                            |
| 68      | 26             | SOCPERH_ON_ULP_GPIO_4  | GPIO_68               | SIO_4                 | USART1_TX             | QEI_IDX               |                       |                       | UART2_RX              | SCT_OUT_0             | PWM_3L                | SCT_IN_0              | PWM_FAULTA                 | USART1_RI                  | PWM_2L                     | SCT_OUT_4                  |
| 69      | 27             | SOCPERH_ON_ULP_GPIO_5  | GPIO_69               | SIO_5                 | USART1_RTS            | QEI_PHA               |                       |                       | UART2_TX              | SCT_OUT_1             | PWM_3H                | SCT_IN_1              | PWM_FAULTB                 | USART1_RS485_EN            | PWM_2H                     | SCT_OUT_5                  |
| 70      | 28             | SOCPERH_ON_ULP_GPIO_6  | GPIO_70               | SIO_6                 | USART1_CTS            | QEI_PHB               | USART1_RX             | I2C2_SCL              | UART2_RTS             | SCT_OUT_2             | PWM_4L                | SCT_IN_2              | PWM_TMR_EXT_TRIG_1         | USART1_RS485_RE            | PMU_TEST_1                 | SCT_OUT_6                  |
| 71      | 29             | SOCPERH_ON_ULP_GPIO_7  | GPIO_71               | SIO_7                 | USART1_IR_RX          | QEI_DIR               | USART1_TX             | I2C2_SDA              | UART2_CTS             | SCT_OUT_3             | PWM_4H                | SCT_IN_3              | PWM_TMR_EXT_TRIG_2         | USART1_RS485_DE            | PMU_TEST_2                 | SCT_OUT_7                  |
| 72      | 30             | SOCPERH_ON_ULP_GPIO_8  | GPIO_72               | SIO_0                 | USART1_IR_TX          | QEI_IDX               |                       |                       | UART2_RX              | SCT_OUT_4             | PWM_SLP_EVENT_TRIG    | UART2_RTS             | PWM_TMR_EXT_TRIG_3         |                            |                            |                            |
| 73      | 31             | SOCPERH_ON_ULP_GPIO_9  | GPIO_73               | SIO_1                 | USART1_RS485_EN       | QEI_PHA               |                       |                       | UART2_TX              | SCT_OUT_5             | PWM_FAULTA            | UART2_CTS             | PWM_TMR_EXT_TRIG_4         |                            |                            |                            |
| 74      | 32             | SOCPERH_ON_ULP_GPIO_10 | GPIO_74               | SIO_2                 | USART1_RS485_RE       | QEI_PHB               | I2C1_SDA              |                       | UART2_RS485_RE        | SCT_OUT_6             | PWM_FAULTB            | UART2_RX              | PMU_TEST_1                 |                            |                            |                            |
| 75      | 33             | SOCPERH_ON_ULP_GPIO_11 | GPIO_75               | SIO_3                 | USART1_RS485_DE       | QEI_DIR               | I2C1_SCL              |                       | UART2_RS485_DE        | SCT_OUT_7             | PWM_TMR_EXT_TRIG_1    | UART2_TX              | PMU_TEST_2                 |                            |                            |                            |


**Table 2:**

Warning: mode 7 (analog) is not supported through the table 1.1 modes 9 and mode 11 interconnections. Only the digital peripheral can be muxed.

| PIN | ULP_GPIO | ULP_GPIO Mode 0 | ULP_GPIO Mode 1 | ULP_GPIO Mode 2 | ULP_GPIO Mode 3 | ULP_GPIO Mode 4 | ULP_GPIO Mode 5 | ULP_GPIO Mode 6 | ULP_GPIO Mode 7 | ULP_GPIO Mode 8 | ULP_GPIO Mode 9 | ULP_GPIO Mode 10 | ULP_GPIO Mode 11 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 0 | ULP_GPIO_0 | ULP_EGPIO[0] | ULP_SPI_CLK | ULP_I2S_DIN | ULP_UART_RTS | ULP_I2C_SDA |  | SOCPERH_ON_ULP_GPIO_0 | AGPIO_0 |  |  |  |  |
| 1 | ULP_GPIO_1 | ULP_EGPIO[1] | ULP_SPI_DOUT | ULP_I2S_DOUT | ULP_UART_CTS | ULP_I2C_SCL | Timer2 | SOCPERH_ON_ULP_GPIO_1 | AGPIO_1 |  |  |  |  |
| 2 | ULP_GPIO_2 | ULP_EGPIO[2] | ULP_SPI_DIN | ULP_I2S_WS | ULP_UART_RX | NPSS_GPIO_4 | COMP1_OUT | SOCPERH_ON_ULP_GPIO_2 | AGPIO_2 |  |  |  |  |
| 4 | ULP_GPIO_4 | ULP_EGPIO[4] | ULP_SPI_CS1 | ULP_I2S_WS | ULP_UART_RTS | ULP_I2C_SDA | AUX_ULP_TRIG_1 | SOCPERH_ON_ULP_GPIO_4 | AGPIO_4 | ULP_SPI_CLK | Timer0 | IR_INPUT |  |
| 5 | ULP_GPIO_5 | ULP_EGPIO[5] | IR_OUTPUT | ULP_I2S_DOUT | ULP_UART_CTS | ULP_I2C_SCL | AUX_ULP_TRIG_0 | SOCPERH_ON_ULP_GPIO_5 | AGPIO_5 | ULP_SPI_DOUT | Timer1 | IR_OUTPUT |  |
| 6 | ULP_GPIO_6 | ULP_EGPIO[6] | ULP_SPI_CS2 | ULP_I2S_DIN | ULP_UART_RX | ULP_I2C_SDA |  | SOCPERH_ON_ULP_GPIO_6 | AGPIO_6 | ULP_SPI_DIN | COMP1_OUT | AUX_ULP_TRIG_0 |  |
| 7 | ULP_GPIO_7 | ULP_EGPIO[7] | IR_INPUT | ULP_I2S_CLK | ULP_UART_TX | ULP_I2C_SCL | Timer1 | SOCPERH_ON_ULP_GPIO_7 | AGPIO_7 | ULP_SPI_CS0 | COMP2_OUT | AUX_ULP_TRIG_1 | NPSS_TEST_MODE_0 |
| 8 | ULP_GPIO_8 | ULP_EGPIO[8] | ULP_SPI_CLK | ULP_I2S_CLK | ULP_UART_CTS | ULP_I2C_SCL | Timer0 | SOCPERH_ON_ULP_GPIO_8 | AGPIO_8 |  |  |  |  |
| 9 | ULP_GPIO_9 | ULP_EGPIO[9] | ULP_SPI_DIN | ULP_I2S_DIN | ULP_UART_RX | ULP_I2C_SDA | NPSS_TEST_MODE_0 | SOCPERH_ON_ULP_GPIO_9 | AGPIO_9 |  |  |  |  |
| 10 | ULP_GPIO_10 | ULP_EGPIO[10] | ULP_SPI_CS0 | ULP_I2S_WS | ULP_UART_RTS | IR_INPUT |  | SOCPERH_ON_ULP_GPIO_10 | AGPIO_10 |  |  |  |  |
| 11 | ULP_GPIO_11 | ULP_EGPIO[11] | ULP_SPI_DOUT | ULP_I2S_DOUT | ULP_UART_TX | ULP_I2C_SDA | AUX_ULP_TRIG_0 | SOCPERH_ON_ULP_GPIO_11 | AGPIO_11 |  |  |  |  |
