idf_build_get_property(target IDF_TARGET)

if(${target} STREQUAL "linux")
    return() # This component is not supported by the POSIX/Linux simulator
endif()

# Always compiled source files
set(srcs)

# Always included headers
set(includes "include"
             "deprecated"
             "i2c/include"
             "ledc/include"
             "parlio/include"
             "rmt/include"
             "sigma_delta/include"
             "temperature_sensor/include"
             "touch_sensor/include"
             "twai/include"
             "uart/include"
             "usb_serial_jtag/include")

# Always included linker fragments
set(ldfragments "linker.lf")

# ADC related source files (dprecated)
if(CONFIG_SOC_ADC_SUPPORTED)
    list(APPEND srcs "deprecated/adc_legacy.c")
endif()

if(CONFIG_SOC_ADC_DMA_SUPPORTED)
    list(APPEND srcs "deprecated/adc_dma_legacy.c")
endif()

# DAC related source files
if(CONFIG_SOC_DAC_SUPPORTED)
    list(APPEND srcs "deprecated/dac_common_legacy.c"
                     "deprecated/${target}/dac_legacy.c")
endif()

# Parallel IO related source files
if(CONFIG_SOC_PARLIO_SUPPORTED)
    list(APPEND srcs "parlio/parlio_common.c" "parlio/parlio_tx.c")
endif()

# GPTimer legacy driver
if(CONFIG_SOC_GPTIMER_SUPPORTED)
    list(APPEND srcs "deprecated/timer_legacy.c")
endif()

# I2C related source files
if(CONFIG_SOC_I2C_SUPPORTED)
    list(APPEND srcs "i2c/i2c.c"
                     "i2c/i2c_master.c"
                     "i2c/i2c_common.c"
        )
    if(CONFIG_SOC_I2C_SUPPORT_SLAVE)
        list(APPEND srcs "i2c/i2c_slave.c")
    endif()

    list(APPEND ldfragments "i2c/linker.lf")
endif()

# I2S related source files
if(CONFIG_SOC_I2S_SUPPORTED)
    list(APPEND srcs "deprecated/i2s_legacy.c")
endif()

# LEDC related source files
if(CONFIG_SOC_LEDC_SUPPORTED)
    list(APPEND srcs "ledc/ledc.c")

    list(APPEND ldfragments "ledc/linker.lf")
endif()

# MCPWM legacy driver
if(CONFIG_SOC_MCPWM_SUPPORTED)
    list(APPEND srcs "deprecated/mcpwm_legacy.c")
endif()

# PCNT legacy driver
if(CONFIG_SOC_PCNT_SUPPORTED)
    list(APPEND srcs "deprecated/pcnt_legacy.c")
endif()

# RMT related source files
if(CONFIG_SOC_RMT_SUPPORTED)
    list(APPEND srcs "rmt/rmt_common.c"
                     "rmt/rmt_encoder.c"
                     "rmt/rmt_rx.c"
                     "rmt/rmt_tx.c"
                     "deprecated/rmt_legacy.c")
    list(APPEND ldfragments "rmt/linker.lf")
endif()

# Sigma-Delta Modulation related source files
if(CONFIG_SOC_SDM_SUPPORTED)
    list(APPEND srcs "sigma_delta/sdm.c"
                     "deprecated/sigma_delta_legacy.c")
endif()

# Temperature Sensor related source files
if(CONFIG_SOC_TEMP_SENSOR_SUPPORTED)
    list(APPEND srcs "temperature_sensor/temperature_sensor.c"
                     "deprecated/rtc_temperature_legacy.c")
endif()

# Touch Sensor related source files
if(CONFIG_SOC_TOUCH_SENSOR_SUPPORTED)
    list(APPEND srcs "touch_sensor/touch_sensor_common.c"
                     "touch_sensor/${target}/touch_sensor.c")
    list(APPEND includes "touch_sensor/${target}/include")
endif()

# TWAI related source files
if(CONFIG_SOC_TWAI_SUPPORTED)
    list(APPEND srcs "twai/twai.c")

    list(APPEND ldfragments "twai/linker.lf")
endif()

# UART related source files
if(CONFIG_SOC_UART_SUPPORTED)
    list(APPEND srcs "uart/uart.c")

    list(APPEND ldfragments "uart/linker.lf")
endif()

# USB Serial JTAG related source files
if(CONFIG_SOC_USB_SERIAL_JTAG_SUPPORTED)
    list(APPEND srcs "usb_serial_jtag/usb_serial_jtag.c"
                     "usb_serial_jtag/usb_serial_jtag_connection_monitor.c")
endif()

# Other source files
if(${target} STREQUAL "esp32")
    list(APPEND srcs "deprecated/adc_i2s_deprecated.c")
endif()

if(BOOTLOADER_BUILD)
    # Bootloader shall NOT depend on the drivers
    idf_component_register()
else()
    # (REQUIRES cannot hide soc headers, since many arguments in the driver headers are chip-dependent)
    # (Legacy drivers requires `esp_adc`, due to ADC HW resource mutex logics are there.
    # Can be removed together with legacy drivers)
    idf_component_register(SRCS "${srcs}"
                           INCLUDE_DIRS ${includes}
                           PRIV_REQUIRES efuse esp_timer esp_mm
                           REQUIRES esp_pm esp_ringbuf freertos soc hal esp_hw_support
                           # for backward compatibility, the driver component needs to
                           # have a public dependency on other "esp_driver_foo" components
                                    esp_driver_gpio esp_driver_pcnt esp_driver_gptimer esp_driver_spi esp_driver_mcpwm
                                    esp_driver_ana_cmpr esp_driver_i2s esp_driver_sdmmc esp_driver_sdspi esp_driver_sdio
                                    esp_driver_dac
                           LDFRAGMENTS ${ldfragments}
                          )
endif()

# If system needs to monitor USJ connection status, then usb_serial_jtag_connection_monitor object file has to be linked
# to the binary, to allow tick hook to be registered
if(CONFIG_USJ_NO_AUTO_LS_ON_CONNECTION OR
   (CONFIG_SOC_USB_SERIAL_JTAG_SUPPORT_LIGHT_SLEEP AND CONFIG_ESP_CONSOLE_USB_SERIAL_JTAG_ENABLED))
    target_link_libraries(${COMPONENT_LIB} INTERFACE "-u usb_serial_jtag_connection_monitor_include")
endif()
